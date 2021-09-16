---
title: Practice gRPC with opentracing
description: I thought this would be easy
date: 2021-09-16
slug: other/opentracing-grpc
image: img/2021/09/PetitMinou.jpg
categories:
  - grpc
  - exp
---

Recently, I spent a lot of time on playing with `opentracing`, so here are some practical samples. `go` codes is [here](https://github.com/AzusaChino/ficus).

First of all, I will treat a successful http request as a valid `Span`.

## OpenTracing With GoFiber

`gofiber` is a http library written by go, `jaeger` is a commly used tracer, here I will use a middleware to collect all info from a span.

> jaeger uses `uber-trace-id` as default propagation in http headers, the format is like `fmt.Sprintf("%016x%016x:%016x:%016x:%x", c.traceID.High, c.traceID.Low, uint64(c.spanID), uint64(c.parentID), flags)`

### fiberopentracing

```go
type Config struct {
    Tracer          opentracing.Tracer // global tracer
    TransactionName func(ctx *fiber.Ctx) string // generate current span name
    Filter          func(ctx *fiber.Ctx) bool // filter some routes
    Modify          func(ctx *fiber.Ctx, span opentracing.Span) // modify current span
}

// Given DefaultConfig
var DefaultConfig = Config{
    Tracer: opentracing.NoopTracer{}, // using dummy tracer
    // add http.* tags to current span
    Modify: func(ctx *fiber.Ctx, span opentracing.Span) {
        span.SetTag("http.method", ctx.Method()) // GET, POST
        span.SetTag("http.remote_address", ctx.IP())
        span.SetTag("http.path", ctx.Path())
        span.SetTag("http.host", ctx.Hostname())
        span.SetTag("http.url", ctx.OriginalURL())
    },
    TransactionName: func(ctx *fiber.Ctx) string {
        return fmt.Sprintf(`HTTP %s URL: %s`, ctx.Method(), ctx.Path())
    },
}

// configDefault function to return default values
func configDefault(config ...Config) Config {
    // Return default config if no config provided
    if len(config) < 1 {
        return DefaultConfig
    }
    cfg := config[0] // always use first config
    if cfg.Tracer == nil {
        cfg.Tracer = DefaultConfig.Tracer
    }
    if cfg.TransactionName == nil {
        cfg.TransactionName = DefaultConfig.TransactionName
    }
    if cfg.Modify == nil {
        cfg.Modify = DefaultConfig.Modify
    }
    return cfg
}

// basically, use fiberoperacing.New() to initialize the middleware

var (
    // custom tags for goFiber
    httpTag = opentracing.Tag{
        Key:   string(ext.SpanKind),
        Value: "http",
    }
    fiberTag = opentracing.Tag{
        Key:   string(ext.Component),
        Value: "fiber",
    }
)

func New(config Config) func(c *fiber.Ctx) error {
    cfg := configDefault(config)
    return func(c *fiber.Ctx) error {
        if cfg.Filter != nil && cfg.Filter(c) {
            return c.Next()
        }

        var span opentracing.Span
        operationName := cfg.TransactionName(c)
        tracer := cfg.Tracer
        // for every request, try get from parent context
        hdr, ok := metadata.FromIncomingContext(c.UserContext())
        if !ok {
            hdr = metadata.New(nil)
            // no container for header, but we can visit them like below
            c.Request().Header.VisitAll(func(key, value []byte) {
                hdr.Set(util.GetString(key), util.GetString(value))
            })
        }

        // 1. extract metadata from hdr with global tracer using httpHeader extractor (HeaderName: uber-tracer-id)
        // 2. if we get a valid spanCtx, we will treat current request as a child of this spanCtx
        // 3. or we start a new span
        if spanCtx, err := opentracing.GlobalTracer().Extract(opentracing.HTTPHeaders, opentracing.HTTPHeadersCarrier(hdr)); err != nil {
            span = tracer.StartSpan(operationName, httpTag, fiberTag)
        } else {
            span = tracer.StartSpan(operationName, ext.RPCServerOption(spanCtx))
        }

        cfg.Modify(c, span)

        // using defer to make sure span.Finish and deal with server 500
        defer func() {
            status := c.Response().StatusCode()
            ext.HTTPStatusCode.Set(span, uint16(status))
            if status >= fiber.StatusInternalServerError {
                ext.Error.Set(span, true)
            }
            span.Finish()
        }()

        // save to fast-http's userData aka request context for further usage. e.g a mysql query, a gRPC client
        c.Locals("tracer", tracer)
        c.Locals("ctx", opentracing.ContextWithSpan(context.Background(), span))
        return c.Next()
    }
}
```

## OpenTracing With gRPC

Here, I use the gRPC client inside http request, for forming a two phases span.

### fibergrpc

```go
const (
    binHdrSuffix = "-bin"
)

var grpcTag = opentracing.Tag{Key: string(ext.Component), Value: "gRPC"}

// Interceptor for wrap gRPC Client Calls
func ClientInterceptor(tracer opentracing.Tracer) grpc.UnaryClientInterceptor {
    return func(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
        newCtx, clientSpan := newClientSpanFromContext(ctx, tracer, method)
        // add custom tags to client span
        clientSpan.SetTag("grpc.target", cc.Target())
        clientSpan.SetTag("grpc.method", method)
        // invoke grpc call
        err := invoker(newCtx, method, req, reply, cc, opts...)
        // finish client span
        finishClientSpan(clientSpan, err)
        return err
    }
}

// metadataTextMap extends a metadata.MD to be an opentracing textmap
type metadataTextMap metadata.MD

// Set is a opentracing.TextMapReader interface that extracts values.
func (m metadataTextMap) Set(key, val string) {
    // gRPC allows for complex binary values to be written.
    encodedKey, encodedVal := encodeKeyValue(key, val)
    // The metadata object is a multiMap, and previous values may exist, but for opentracing headers, we do not append
    // we just override.
    m[encodedKey] = []string{encodedVal}
}

// ForeachKey is a opentracing.TextMapReader interface that extracts values.
func (m metadataTextMap) ForeachKey(callback func(key, val string) error) error {
    for k, vv := range m {
        for _, v := range vv {
            if err := callback(k, v); err != nil {
                return err
            }
        }
    }
    return nil
}

// encodeKeyValue encodes key and value qualified for transmission via gRPC.
func encodeKeyValue(k, v string) (string, string) {
    k = strings.ToLower(k)
    if strings.HasSuffix(k, binHdrSuffix) {
        val := base64.StdEncoding.EncodeToString([]byte(v))
        v = val
    }
    return k, v
}

type clientSpanTagKey struct{}

func newClientSpanFromContext(ctx context.Context, tracer opentracing.Tracer, operateName string) (context.Context, opentracing.Span) {
    var parentSpanCtx opentracing.SpanContext
    // fetch possible existed parent context which withValue of span from ctx
    if parent := opentracing.SpanFromContext(ctx); parent != nil {
        parentSpanCtx = parent.Context()
    }
    opts := []opentracing.StartSpanOption{
        opentracing.ChildOf(parentSpanCtx),
        ext.SpanKindRPCClient,
        grpcTag,
    }
    // check ctx, if contains StartSpanOption
    if tag := ctx.Value(clientSpanTagKey{}); tag != nil {
        if opt, ok := tag.(opentracing.StartSpanOption); ok {
            opts = append(opts, opt)
        }
    }

    clientSpan := tracer.StartSpan(operateName, opts...)

    // create new textMap carrier
    md := metadataTextMap{}
    // inject data from current span to textMap carrier
    if err := tracer.Inject(clientSpan.Context(), opentracing.HTTPHeaders, md); err != nil {
        // record inject error, can't panic, will break the span
        clientSpan.LogFields(log.String("inject-error", err.Error()))
    }
    // set metadata to ctx
    ctxWithMetadata := metadata.NewOutgoingContext(ctx, metadata.MD(md))
    return opentracing.ContextWithSpan(ctxWithMetadata, clientSpan), clientSpan
}

func finishClientSpan(clientSpan opentracing.Span, err error) {
    if err != nil && err != io.EOF {
        ext.Error.Set(clientSpan, true)
        clientSpan.LogFields(log.String("invoke-error", err.Error()))
    }
    clientSpan.Finish()
}
```

### hello.go (sample usage)

```go
func DoHello(msg string, ctx *fasthttp.RequestCtx) string {
    var addr = fmt.Sprintf("%s:%s", conf.GrpcConfig.Server, conf.GrpcConfig.Port)
    // fetch custom values from ctx.Locals(), which were passed from gofiber interceptor
    c := ctx.UserValue("ctx").(context.Context)
    tracer := ctx.UserValue("tracer").(opentracing.Tracer)
    // wrap grpc client with interceptor, and I think this c will be the first parameter in grpc.UnaryClientInterceptor
    conn, err := grpc.DialContext(c, addr, grpc.WithInsecure(), grpc.WithBlock(),
        grpc.WithUnaryInterceptor(rpc.ClientInterceptor(tracer)))
    if err != nil {
        log.Panic(err)
    }
    defer func(conn *grpc.ClientConn) {
        err := conn.Close()
        if err != nil {
            log.Panic(err)
        }
    }(conn)

    client := pb.NewHelloServiceClient(conn)
    // do grpc call
    r, err := client.SayHello(c, &pb.Request{
        Id:   uuid.New().String(),
        Msg:  msg,
        Date: time.Now().Format("20060102"),
    })
    if err != nil {
        log.Panic(err)
    }
    log.Printf("Hello From GRPC server, code: %v, msg: %s", r.Code, r.Msg)
    return r.String()
}
```

## Opentracing with java grpc (server)

Similarly, We will use interceptor to intercept all connections.

### ServerTracingInterceptor

```java
public class CustomServerTracingInterceptor implements ServerInterceptor {

    private final Tracer tracer;
    private final OperationNameConstructor operationNameConstructor;
    private final boolean streaming; // wait for further learning
    private final boolean verbose; // wait for further learning
    private final Set<ServerRequestAttribute> tracedAttributes;

    public CustomServerTracingInterceptor(Tracer tracer) {
        this.tracer = tracer;
        this.operationNameConstructor = OperationNameConstructor.DEFAULT;
        this.streaming = false;
        this.verbose = false;
        this.tracedAttributes = new HashSet<>(Arrays.asList(ServerRequestAttribute.HEADERS,
                ServerRequestAttribute.METHOD_NAME, ServerRequestAttribute.METHOD_TYPE, ServerRequestAttribute.CALL_ATTRIBUTES));
    }

    private CustomServerTracingInterceptor(Tracer tracer, OperationNameConstructor operationNameConstructor, boolean streaming,
                                           boolean verbose, Set<ServerRequestAttribute> tracedAttributes) {
        this.tracer = tracer;
        this.operationNameConstructor = operationNameConstructor;
        this.streaming = streaming;
        this.verbose = verbose;
        this.tracedAttributes = tracedAttributes;
    }

    public ServerServiceDefinition intercept(ServerServiceDefinition serviceDef) {
        return ServerInterceptors.intercept(serviceDef, this);
    }

    /**
     * Add tracing to all requests made to this service.
     *
     * @param bindableService to intercept
     * @return the serviceDef with a tracing interceptor
     */
    public ServerServiceDefinition intercept(BindableService bindableService) {
        return ServerInterceptors.intercept(bindableService, this);
    }

    /**
     * Intercept {@link ServerCall} dispatch by the {@code next} {@link ServerCallHandler}. General
     * semantics of {@link ServerCallHandler#startCall} apply and the returned
     * {@link ServerCall.Listener} must not be {@code null}.
     *
     * <p>If the implementation throws an exception, {@code call} will be closed with an error.
     * Implementations must not throw an exception if they started processing that may use {@code
     * call} on another thread.
     *
     * @param call    object to receive response messages
     * @param headers which can contain extra call metadata from {@link ClientCall#start},
     *                e.g. authentication credentials.
     * @param next    next processor in the interceptor chain
     * @return listener for processing incoming messages for {@code call}, never {@code null}.
     */
    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(ServerCall<ReqT, RespT> call, Metadata headers, ServerCallHandler<ReqT, RespT> next) {
        // translate headers
        Map<String, String> headerMap = new HashMap<>();
        for (String k : headers.keys()) {
            if (!k.endsWith(Metadata.BINARY_HEADER_SUFFIX)) {
                String v = headers.get(Metadata.Key.of(k, Metadata.ASCII_STRING_MARSHALLER));
                headerMap.put(k, v);
            }
        }
        final String operateName = this.operationNameConstructor.constructOperationName(call.getMethodDescriptor());
        final Span span = this.getSpanFromHeaders(headerMap, operateName);

        // add tags
        for (CustomServerTracingInterceptor.ServerRequestAttribute attr : this.tracedAttributes) {
            switch (attr) {
                case METHOD_TYPE:
                    span.setTag("grpc.method_type", call.getMethodDescriptor().getType().toString());
                    break;
                case METHOD_NAME:
                    span.setTag("grpc.method_name", call.getMethodDescriptor().getFullMethodName());
                    break;
                case CALL_ATTRIBUTES:
                    span.setTag("grpc.call_attributes", call.getAttributes().toString());
                    break;
                case HEADERS:
                    span.setTag("grpc.headers", headers.toString());
                    break;
                default:
                    break;
            }
        }
        // pass span to current context
        Context ctxWithSpan = Context.current().withValue(OpenTracingContextKey.getKey(), span);
        ServerCall.Listener<ReqT> listenerWithContext = Contexts.interceptCall(ctxWithSpan, call, headers, next);

        // deal with all hooks, and add tags in every situation
        return new ForwardingServerCallListener.SimpleForwardingServerCallListener<>(listenerWithContext) {

            @Override
            public void onMessage(ReqT message) {
                if (streaming || verbose) {
                    span.log(ImmutableMap.of("Message received", message));
                }
                delegate().onMessage(message);
            }

            @Override
            public void onHalfClose() {
                if (streaming) {
                    span.log("Client finished sending messages");
                }
                delegate().onHalfClose();
            }

            @Override
            public void onCancel() {
                span.log("Call cancelled");
                span.finish();
                delegate().onCancel();
            }

            @Override
            public void onComplete() {
                if (verbose) {
                    span.log("Call completed");
                }
                span.finish();
                delegate().onComplete();
            }
        };
    }

    /**
     * generate span info by using headers
     */
    private Span getSpanFromHeaders(Map<String, String> headers, String operateName) {
        Tracer.SpanBuilder spanBuilder;
        try {
            // extract spanCtx information from headers (by using HeaderName: uber-trace-id in default)
            SpanContext parentSpanCtx = this.tracer.extract(Format.Builtin.HTTP_HEADERS, new TextMapAdapter(headers));
            if (Objects.isNull(parentSpanCtx)) {
                spanBuilder = this.tracer.buildSpan(operateName);
            } else {
                spanBuilder = this.tracer.buildSpan(operateName).asChildOf(parentSpanCtx);
            }
        } catch (IllegalArgumentException e) {
            spanBuilder = this.tracer.buildSpan(operateName)
                    .withTag("Error", "Extract failed and an IllegalArgumentException was thrown");
        }
        return spanBuilder.start();
    }

    public enum ServerRequestAttribute {
        /**
         * Request Attr, for adding tags
         */
        HEADERS,
        METHOD_TYPE,
        METHOD_NAME,
        CALL_ATTRIBUTES
    }
}
```

### HelloGrpcServer (sample usage)

```java
    public void start() throws IOException {
        CustomServerTracingInterceptor tracingInterceptor = new CustomServerTracingInterceptor(this.tracer);
        int port = 50051;
        server = ServerBuilder.forPort(port)
        // intercept current server
            .addService(tracingInterceptor.intercept(new HelloImpl()))
            .build()
            .start();
        LOGGER.info("server started, listening on " + port);
        Runtime.getRuntime().addShutdownHook(new ThreadFactoryBuilder().build().newThread(
            () -> {
                LOGGER.warn("shutting down gRPC server since JVM is shutting down.");
                try {
                    HelloGrpcServer.this.stop();
                } catch (InterruptedException e) {
                    e.printStackTrace(System.err);
                }
                LOGGER.warn("*** server shut down");
            }
        ));
    }
    static class HelloImpl extends HelloServiceGrpc.HelloServiceImplBase {
        @Override
        public void sayHello(Request request, StreamObserver<Response> responseObserver) {
            Response response = Response.newBuilder().setMsg("Hello " + request.getMsg() + ", today is " + request.getDate()).setCode(200).build();
            responseObserver.onNext(response);
            responseObserver.onCompleted();
        }
    }
```

## Temporary Conclusion

1. First of all, we need to understand what is a `Span`. In web applications, a successful request can be a valid span, still, we can split this request into several small spans to analyze which part is the lower bound.
2. And, we can see that all tracing are done inside interceptors, just like aop. We can do a lot of works with interceptors.

## References

- [grpc-opentracing](https://github.com/grpc-ecosystem/grpc-opentracing)
- [fiber-opentracing](https://github.com/aschenmaker/fiber-opentracing)
