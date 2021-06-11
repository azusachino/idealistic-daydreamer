---
title: Java读取文件的几种方式
description: 最终还是怎么方便，怎么来
date: 2021-05-31
slug: java/file
image: img/2021/05/miku-pan.jpg
categories:
  - exp
  - java
---

近期开发涉及到了文件的读取，故重温了常用的几种方式。

```java
public class FileRead {

    private static final Logger LOGGER = LoggerFactory.getLogger(FileRead.class);

    private static final Charset UTF8 = StandardCharsets.UTF_8;

    // 回车符
    private static final int LF = 10;

    // 换行符
    private static final int CR = 13;

    public static void main(String[] args) {
        String file = "C:\\ProgramData\\chocolatey\\logs\\chocolatey.log";
        StopWatch stopWatch = StopWatch.create("1");

        // run(() -> byteRead(file), stopWatch);
        run(() -> bufferedRead(file), stopWatch);
        run(() -> streamRead(file), stopWatch);
        run(() -> scannerRead(file), stopWatch);
        run(() -> readLine(file), stopWatch);

        LOGGER.info(stopWatch.prettyPrint());
    }

    /**
     * 按字节读取
     */
    // public static void byteRead(String file) {
    //     try (InputStream is = new FileInputStream(new File(file))) {
    //         while (is.read() != -1) {
    //             // do something
    //         }
    //     } catch (Exception e) {

    //         LOGGER.error("错误", e);
    //     }
    // }

    /**
     * BufferReader按行读取
     */
    public static void bufferedRead(String file) {
        try (BufferedReader br = new BufferedReader(new FileReader(file))) {
            while (br.readLine() != null) {
                // do something
            }
        } catch (IOException e) {
            LOGGER.error("错误", e);
        }
    }

    /**
     * Stream按行读取
     */
    public static void streamRead(String file) {
        try (Stream<String> stream = Files.lines(Paths.get(file))) {
        } catch (IOException e) {
            LOGGER.error("错误", e);
        }
    }

    /**
     * Scanner按行读取
     */
    public static void scannerRead(String file) {
        try (Scanner scan = new Scanner(new File(file))) {
            while (scan.hasNextLine()) {
                scan.nextLine();
            }
        } catch (IOException e) {
            LOGGER.error("错误", e);
        }
    }

    /**
     * NIO按行读取
     */
    public static void readLine(String fileName) {

        try (RandomAccessFile randomAccessFile = new RandomAccessFile(fileName, "rw");
                FileChannel channel = randomAccessFile.getChannel()) {
            // 每次读取的buffer
            ByteBuffer buffer = ByteBuffer.allocate(1024 * 1024);
            // 期望每行占用的buffer
            ByteBuffer stringBuffer = ByteBuffer.allocate(20);
            // 从文件中读数据到buffer中
            while (channel.read(buffer) != -1) {

                // 切换模式，写->读
                buffer.flip();

                while (buffer.hasRemaining()) {
                    byte b = buffer.get();
                    // 换行或回车
                    if (b == LF || b == CR) {
                        stringBuffer.flip();
                        // 解码已经读到的一行所对应的字节
                        UTF8.decode(stringBuffer).toString();
                        // System.out.println(line + "----------");
                        stringBuffer.clear();
                    } else {
                        // 空间不够扩容
                        if (!stringBuffer.hasRemaining()) {
                            stringBuffer = reAllocate(stringBuffer);
                        }
                        stringBuffer.put(b);
                    }
                }
                // 清空,position位置为0，limit=capacity
                buffer.clear();
            }
        } catch (Exception e) {
            LOGGER.error("错误", e);
        }
    }

    private static ByteBuffer reAllocate(ByteBuffer stringBuffer) {
        final int capacity = stringBuffer.capacity();
        // 扩容至两倍
        byte[] newBuffer = new byte[capacity * 2];
        System.arraycopy(stringBuffer.array(), 0, newBuffer, 0, capacity);
        // 生成新的ByteBuffer，并移动到position
        return (ByteBuffer) ByteBuffer.wrap(newBuffer).position(capacity);
    }

    private static void run(Runnable r, StopWatch sw) {
        sw.start();
        r.run();
        sw.stop();
    }
}
```

## 读取一个 10M 的日志文件

简单地比较之后，Stream 的数据最好看，而且代码很简单，所以就采取这个方案了。

```output
14:59:04.292 [main] INFO cn.az.code.file.FileRead - StopWatch '1': running time = 703973100 ns
---------------------------------------------
ns         %     Task name
---------------------------------------------
077901600  011%
013554200  002%
440220500  063%
172296800  024%
```

## 参考链接

- [使用 NIO——FileChannel 按行读取文件](https://www.codeleading.com/article/16182950860/)
