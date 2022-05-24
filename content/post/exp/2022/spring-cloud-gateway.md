---
title: Spring Cloud Gateway 实践
description: Order is important
date: 2022-05-24
slug: spring-cloud-gateway
image: img/2022/05/GlassBridge.jpg
categories:
  - exp
tags:
  - java
  - spring
  - webflux
---

论如何在单个 Web 项目中集成 Spring Cloud Gateway 的业务逻辑.

## 项目依赖

```xml
<dependencies>
    <!-- Web [Netty] 基础能力 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>
    <!-- Gateway 相关能力 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-gateway</artifactId>
    </dependency>
    <!-- Load Balance 相关能力 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-loadbalancer</artifactId>
    </dependency>
</dependencies>
```

## 业务链路

![.](img/2022/05/gateway-flow-chart.svg)

### `RouterFunctionMapping`

默认优先级最高的 `HandlerMapping`. 若定义了一些 `RouterFunction`, 将会被最优先匹配; 如:

```java
    @Bean
    public RouterFunction<ServerResponse> routerFunction() {
        return RouterFunctions.route(RequestPredicates.GET("/api/v1/gateway"), r -> ServerResponse.ok().bodyValue("ok"));
    }
```

### `RequestMappingHandlerMapping`

Web 框架中的核心 `HandlerMapping`, 用于扫描所有的 `@RequestMapping` 方法; 核心业务逻辑如下:

```java
    /**
     * Look up the best-matching handler method for the current request.
     * If multiple matches are found, the best match is selected.
     * @param exchange the current exchange
     * @return the best-matching handler method, or {@code null} if no match
     * @see #handleMatch
     * @see #handleNoMatch
     */
    @Nullable
    protected HandlerMethod lookupHandlerMethod(ServerWebExchange exchange) throws Exception {
        List<Match> matches = new ArrayList<>();
        // 直接根据请求路径进行匹配
        List<T> directPathMatches = this.mappingRegistry.getMappingsByDirectPath(exchange);
        if (directPathMatches != null) {
            addMatchingMappings(directPathMatches, matches, exchange);
        }
        if (matches.isEmpty()) {
            addMatchingMappings(this.mappingRegistry.getRegistrations().keySet(), matches, exchange);
        }
        // 如果有多条匹配, 选择一个最优的
        if (!matches.isEmpty()) {
            Comparator<Match> comparator = new MatchComparator(getMappingComparator(exchange));
            matches.sort(comparator);
            Match bestMatch = matches.get(0);
            if (matches.size() > 1) {
                if (logger.isTraceEnabled()) {
                    logger.trace(exchange.getLogPrefix() + matches.size() + " matching mappings: " + matches);
                }
                if (CorsUtils.isPreFlightRequest(exchange.getRequest())) {
                    for (Match match : matches) {
                        if (match.hasCorsConfig()) {
                            return PREFLIGHT_AMBIGUOUS_MATCH;
                        }
                    }
                }
                else {
                    Match secondBestMatch = matches.get(1);
                    if (comparator.compare(bestMatch, secondBestMatch) == 0) {
                        Method m1 = bestMatch.getHandlerMethod().getMethod();
                        Method m2 = secondBestMatch.getHandlerMethod().getMethod();
                        RequestPath path = exchange.getRequest().getPath();
                        throw new IllegalStateException(
                                "Ambiguous handler methods mapped for '" + path + "': {" + m1 + ", " + m2 + "}");
                    }
                }
            }
            handleMatch(bestMatch.mapping, bestMatch.getHandlerMethod(), exchange);
            return bestMatch.getHandlerMethod();
        }
        else {
            return handleNoMatch(this.mappingRegistry.getRegistrations().keySet(), exchange);
        }
    }
```

### `RoutePredicateHandlerMapping`

Spring Cloud Gateway 中的核心组件, 用于拦截请求, 并根据用户配置的 Predicate 条件进行匹配; 若当前请求满足某条规则; 则返回 `FilteringWebHandler` 实例，并按照 `Ordered` 顺序依次执行 `GlobalFilter` 的实现类。

## 实现一个 `GlobalFilter`

```java
/**
 * 自定义的负载均衡拦截器
 */
public class TsubameGatewayFilter implements GlobalFilter, Ordered {

    private static final Logger LOGGER = LoggerFactory.getLogger(TsubameGatewayFilter.class);

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        LOGGER.info("进入 TsubameGatewayFilter");
        URI uri = exchange.getRequest().getURI();
        ServiceInstance si = ServiceInstanceHolder.get("tsubame-gateway");
        DelegatingServiceInstance delegatingServiceInstance = new DelegatingServiceInstance(si, "http");
        URI newUri = LoadBalancerUriTools.reconstructURI(delegatingServiceInstance, uri);
        // 构造出负载均衡后的目标地址, 并设置到 exchange.attributes 中
        // 执行顺序倒数第二的 NettyRoutingFilter 会读取此参数，并调用 WebClient 请求相应的地址
        exchange.getAttributes().put(ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR, newUri);
        // 向后续 GlobalFilter 传递 exchange
        return chain.filter(exchange.mutate().request(exchange.getRequest().mutate().header("x-load-balanced", "true").build()).build());
    }

    @Override
    public int getOrder() {
        // 先于 spring-cloud-starter-loadbalancer 自带的负载均衡拦截器执行
        return ReactiveLoadBalancerClientFilter.LOAD_BALANCER_CLIENT_FILTER_ORDER - 1;
    }
}
```

## 保证 `AbstractHandlerMapping` 实现类的执行顺序

由于本次想要实现的是, 在单个 Web 应用中集成 Spring Cloud Gateway 的功能, 那么就必须要保证 `RoutePredicateHandlerMapping` 在 `RequestMappingHandlerMapping` 前执行. 在 `GatewayAutoConfiguration` 中观察到, 注册 `RoutePredicateHandlerMapping` 的时候标注了 `ConditionalOnMissingBean`, 所以我们可以覆写 `Bean` 注册, 同时调整 Order 值.

```java
    @Bean
    public RoutePredicateHandlerMapping routePredicateHandlerMapping(FilteringWebHandler webHandler, RouteLocator routeLocator, GlobalCorsProperties globalCorsProperties, Environment environment) {
        OneShotRoutePredicateHandlerMapping routePredicateHandlerMapping = new OneShotRoutePredicateHandlerMapping(webHandler, routeLocator, globalCorsProperties, environment);
        // RequestMappingHandlerMapping.order(0);
        routePredicateHandlerMapping.setOrder(-1);
        return routePredicateHandlerMapping;
    }
```

## 保证 `FilteringWebHandler` 仅执行一次

如果不做任何处理的话, 请求将永远无法达到真实服务, 因为网关始终会拦截并处理; 因此需要在处理前做判断, 如果包含特殊标识, 直接在 `getHandlerInternal` 方法中返回 `Mono.empty()` 即可, `DispatcherHandler` 将会遍历下一个 `HandlerMapping` 以找到合适的 `HandlerMappingHandler`.

```java
/**
 * 仅拦截一次的网关 HandlerMapping
 */
public class OneShotRoutePredicateHandlerMapping extends RoutePredicateHandlerMapping {

    public OneShotRoutePredicateHandlerMapping(FilteringWebHandler webHandler, RouteLocator routeLocator, GlobalCorsProperties globalCorsProperties, Environment environment) {
        super(webHandler, routeLocator, globalCorsProperties, environment);
    }

    @Override
    protected Mono<?> getHandlerInternal(ServerWebExchange exchange) {
        // 在实现自定义的负载均衡拦截器时，在最后的请求信息中添加了自定义的头 `x-load-balanced`
        if (StringUtils.hasText(exchange.getRequest().getHeaders().getFirst("x-load-balanced"))) {
            // 返回 Mono.empty() 以触发下一个 HandlerMapping 的执行
            return Mono.empty();
        }
        // 原始的拦截业务逻辑
        return super.getHandlerInternal(exchange);
    }
}
```

## 结语

一开始接触到 Spring Cloud Gateway 真的是不知道从哪里下手，但是随着时间渐渐地推移，加上很多资料和源码的熟悉，这方面的执行链路也变得更加清晰起来；虽然花了接近一周时间在泥坑里乱爬，但后续花一天时间就完成了剩余的工作；这也是一种成长吧。
