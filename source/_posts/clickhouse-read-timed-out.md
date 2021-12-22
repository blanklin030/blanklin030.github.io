---
title: ClickHouse Read Timed Out追踪过程
date: 2021-12-21 19:28:55
tags:
  - clickhouse
  - http
  - tcp
categories:
  - bigdata
---
### 问题背景
最近经常遇到`Read timed out`报错，具体内容看下面，从字面意思判断我们知道是读超时，那为什么会发生读超时呢？
```
ru.yandex.clickhouse.except.ClickHouseException: ClickHouse exception, code: 159, host: bigdata-clickhouse-xxxx.ys, port: 8023; Read timed out	at ru.yandex.clickhouse.except.ClickHouseExceptionSpecifier.getException(ClickHouseExceptionSpecifier.java:86)	at ru.yandex.clickhouse.except.ClickHouseExceptionSpecifier.specify(ClickHouseExceptionSpecifier.java:56)	at ru.yandex.clickhouse.except.ClickHouseExceptionSpecifier.specify(ClickHouseExceptionSpecifier.java:25)	at ru.yandex.clickhouse.ClickHouseStatementImpl.getInputStream(ClickHouseStatementImpl.java:797)	at ru.yandex.clickhouse.ClickHouseStatementImpl.getLastInputStream(ClickHouseStatementImpl.java:691)	at ru.yandex.clickhouse.ClickHouseStatementImpl.executeQuery(ClickHouseStatementImpl.java:340)	at ru.yandex.clickhouse.ClickHouseStatementImpl.executeQuery(ClickHouseStatementImpl.java:324)	at ru.yandex.clickhouse.ClickHouseStatementImpl.executeQuery(ClickHouseStatementImpl.java:319)	at ru.yandex.clickhouse.ClickHouseStatementImpl.executeQuery(ClickHouseStatementImpl.java:314)	at 
```
### 源码解读
+ 从抛出的堆栈入手，先读`ru.yandex.clickhouse.except.ClickHouseExceptionSpecifier`类
```
 /**
  * Here we expect the ClickHouse error message to be of the following format:
  * "Code: 10, e.displayText() = DB::Exception: ...".
  */
private static ClickHouseException specify(String clickHouseMessage, Throwable cause, String host, int port) {
    if (Utils.isNullOrEmptyString(clickHouseMessage) && cause != null) {
        return getException(cause, host, port);
    }

    try {
        int code;
        if (clickHouseMessage.startsWith("Poco::Exception. Code: 1000, ")) {
            code = 1000;
        } else {
            // Code: 175, e.displayText() = DB::Exception:
            code = getErrorCode(clickHouseMessage);
        }
        // ошибку в изначальном виде все-таки укажем
        Throwable messageHolder = cause != null ? cause : new Throwable(clickHouseMessage);
        if (code == -1) {
            return getException(messageHolder, host, port);
        }

        return new ClickHouseException(code, messageHolder, host, port);
    } catch (Exception e) {
        log.error("Unsupported ClickHouse error format, please fix ClickHouseExceptionSpecifier, message: {}, error: {}", clickHouseMessage, e.getMessage());
        return new ClickHouseUnknownException(clickHouseMessage, cause, host, port);
    }
}
private static ClickHouseException getException(Throwable cause, String host, int port) {
        if (cause instanceof SocketTimeoutException)
        // if we've got SocketTimeoutException, we'll say that the query is not good. This is not the same as SOCKET_TIMEOUT of clickhouse
        // but it actually could be a failing ClickHouse
        {
            return new ClickHouseException(ClickHouseErrorCode.TIMEOUT_EXCEEDED.code, cause, host, port);
        } else if (cause instanceof ConnectTimeoutException || cause instanceof ConnectException)
        // couldn't connect to ClickHouse during connectTimeout
        {
            return new ClickHouseException(ClickHouseErrorCode.NETWORK_ERROR.code, cause, host, port);
        } else {
            return new ClickHouseUnknownException(cause, host, port);
        }
    }
```
注意到上文中异常类型是`SocketTimeoutException`，为啥会是这个异常呢？
+ 继续阅读`ru.yandex.clickhouse.ClickHouseStatementImpl`类
```
private InputStream getInputStream(
  ClickHouseSqlStatement parsedStmt,
  Map<ClickHouseQueryParam, String> additionalClickHouseDBParams,
  List<ClickHouseExternalData> externalData,
  Map<String, String> additionalRequestParams
) throws ClickHouseException {
  .............................
  HttpEntity entity = null;
  try {
      uri = followRedirects(uri);
      HttpPost post = new HttpPost(uri);
      post.setEntity(requestEntity);

      if (parsedStmt.isIdemponent()) {
          httpContext.setAttribute("is_idempotent", Boolean.TRUE);
      } else {
          httpContext.removeAttribute("is_idempotent");
      }
      
      HttpResponse response = client.execute(post, httpContext);
      entity = response.getEntity();
      checkForErrorAndThrow(entity, response);

      InputStream is;
      if (entity.isStreaming()) {
          is = entity.getContent();
      } else {
          FastByteArrayOutputStream baos = new FastByteArrayOutputStream();
          entity.writeTo(baos);
          is = baos.convertToInputStream();
      }

      // retrieve response summary
      if (isQueryParamSet(ClickHouseQueryParam.SEND_PROGRESS_IN_HTTP_HEADERS, additionalClickHouseDBParams, additionalRequestParams)) {
          Header summaryHeader = response.getFirstHeader("X-ClickHouse-Summary");
          currentSummary = summaryHeader != null ? Jackson.getObjectMapper().readValue(summaryHeader.getValue(), ClickHouseResponseSummary.class) : null;
      }

      return is;
  } catch (ClickHouseException e) {
      throw e;
  } catch (Exception e) {
      log.info("Error during connection to {}, reporting failure to data source, message: {}", properties, e.getMessage());
      EntityUtils.consumeQuietly(entity);
      log.info("Error sql: {}", sql);
      throw ClickHouseExceptionSpecifier.specify(e, properties.getHost(), properties.getPort());
  }
}
```
注意到这是建立一个`http`连接，使用`post`方式，等到数据传输完成，返回执行结果，如果有任何`ClickHouseException`则直接抛掉，否则捕获未知异常`Exception`，格式化成固定文本的错误信息，这个错误信息就是在`ClickHouseExceptionSpecifier.specify`方法里被处理成`Code: 10, e.displayText() = DB::Exception: ...`这种结构。
所以回到上面的结论，就是说这个`http`连接出现异常，未等到结果，抛出了`SocketTimeoutException`，那么根据这个异常，我们可以定位到根本原因了
+ 接着看`java.net.SocketTimeoutException`类
```
/**
 * Signals that a timeout has occurred on a socket read or accept.
 *
 * @since   1.4
 */

public class SocketTimeoutException extends java.io.InterruptedIOException {
    private static final long serialVersionUID = -8846654841826352300L;

    /**
     * Constructs a new SocketTimeoutException with a detail
     * message.
     * @param msg the detail message
     */
    public SocketTimeoutException(String msg) {
        super(msg);
    }

    /**
     * Construct a new SocketTimeoutException with no detailed message.
     */
    public SocketTimeoutException() {}
}
```
这个类非常简短，但是很清晰告诉我们，会抛出`SocketTimeoutException`异常的原因只能是一个`socket`读取或者接收时发生了超时
+ 题外话什么是`socket`
`socket`的含义就是两个应用程序通过一个双向的通信连接实现数据的交换，连接的一段就是一个`socket`，又称为套接字。实现一个`socket`连接通信至少需要两个套接字，一个运行在服务端（插孔），一个运行在客户端（插头）。套接字用于描述`IP`地址和端口，是一个通信链的句柄。应用程序通过套接字向网络发出请求或应答网络请求。注意的是套接字既不是程序也不是协议，只是操作系统提供给通信层的一组抽象`API`接口。
`socket`是应用层与`TCP/IP`协议簇通信的中间抽象层，是一组接口。在设计模式中其实就是门面模式。`Socket`将复杂的`TCP/IP`协议簇隐藏在接口后面，对于用户而言，一组接口即可让`Socket`去组织数据，以符合指定的协议。
![1](/images/clickhouse/jdbc/1.png)

### 回归正题，什么是Read Timed Out
所谓`read timed out`就是读超时，`http`连接已创建，客户端等待服务端返回结果，等待时间超过了超时时间，所以客户端连接断开，抛出`SocketTimeoutException`异常。我们可以模拟出这个过程，代码如下
```
public class ReadTimedOutTest {

  public static void main(String[] args) {
    try {
      ServerSocket serverSocket = new ServerSocket(8888, 200);
      Thread.sleep(66666666);
    } catch (Exception e) {
      e.printStackTrace();
    }
  }

  @Test
  public void tetReadTimedOut() {
    Socket socket = new Socket();
    long starTime = 0;
    try {
      socket.connect(new InetSocketAddress("127.0.0.1", 8888), 10000);
      System.out.println("socket连接成功。。。。");
      socket.setSoTimeout(2000);
      starTime = System.currentTimeMillis();
      int read = socket.getInputStream().read();
    } catch (Exception e) {
      e.printStackTrace();
    } finally {
      long endTime = System.currentTimeMillis();
      System.out.println("执行时间："+(endTime - starTime));
    }
  }
}
```
报错信息如下截图：
![2](/images/clickhouse/jdbc/2.png)
意思是说已连接服务端的8888端口，握手是正常的，然后开始传输数据，因为限制了服务端`Thread.sleep`，让服务端无法给客户端传输数据

### 解决方法
`clickhouse-jdbc`默认给`SOCKET_TIMEOUT`设置的时间是30s，由于服务端不知道何时能返回结果（此时间受`settings.max_execution_time`影响），所以我们最好给`jdbc`设置一个**socket_timeout=max_execution_time+10s**，防止服务端还在处理，而客户端已经超时断开连接。
![3](/images/clickhouse/jdbc/3.png)

### 总结
+ `Read timed out`表示已经连接成功(即三次握手已经完成)，但是服务器没有及时返回数据(没有在设定的时间内返回数据)，导致读超时。
+ `java`在`linux`中的 `Read timed out` 并不是通过`C`函数`setSockOpt(SO_RCVTIMEO)`来设置的，而是通过`select(s, timeout)`来实现定时器，并抛出`JNI`异常来控制的
+ `java socket`读超时的设置是在`read()`方法被调用的时候传入的，所以只要在`read()`调用之前设置即可
