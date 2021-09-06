# Spring 使用 RequestBodyAdvice 来实现请求参数的加解密预处理



![图片](https://mmbiz.qpic.cn/mmbiz_png/a4Fvz13svYmIiazp7icxCBHcyQibS49CtiatxF82rRxJavDmpQh08elAALpQIkJ6yaG3vXaLXekx1oQMvS3VWb18sQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



## 前言

在我们平常的项目开发中，一般会遇到这样的需求：

1. 对请求参数记录日志
2. 对入参进行解密和验签（在一些金融项目或者安全性要求比较高的项目中经常会出现这样的需求）
3. 对出参进行加密

像打日志这种需求就比较简单了，这里主要说一下第二个问题

## 常见解决方案

针对对上面**对入参进行解密和验签**问题一般可以使用以下几种方案：

1. 使用 `HandlerInterceptor`来做
2. 使用 `HttpMessageConverter` 在消息转换的时候进行加解密操作
3. 使用 `RequestBodyAdvice` 在请求未被 `Controller` 处理前，请请求参数进行加密验签操作
4. 在每个接口方法中单独处理
5. 只写一个接口，在接口中进行加解密，并根据请求参数中某个特定字段来执行不同的逻辑

以上的解决方案都能解决我们的问题，这里不一一介绍每个方案是怎么实现的，主要讲一下 `RequestBodyAdvice` 的使用

# RequestBodyAdvice 介绍

从源码中可以看出：**允许在读取请求的主体并将其转换为请求之前对其进行自定义对象，并且还允许在生成对象之前对其进行处理。**

```java
/**
 * Allows customizing the request before its body is read and converted into an
 * Object and also allows for processing of the resulting Object before it is
 * passed into a controller method as an {@code @RequestBody} or an
 * {@code HttpEntity} method argument.
 */
public interface RequestBodyAdvice {

    /**
     * 第一步被调用：判断当前的拦截器是否支持，如果返回 false, 则该拦截器不处理请求信息
     * Invoked first to determine if this interceptor applies.
     */
    boolean supports(MethodParameter methodParameter, Type targetType,
            Class<? extends HttpMessageConverter<?>> converterType);

    /**
     * 第二步被调用：在读取和转换请求正文之前调用。 
     * Invoked second before the request body is read and converted.
     *
     * 在这里可以通过解析 inputMessage 的 body ，对原 body 进行解密，
     * 将解密的数据重新构建一个 HttpInputMessage，来实现加解密操作
     */
    HttpInputMessage beforeBodyRead(HttpInputMessage inputMessage, MethodParameter parameter,
            Type targetType, Class<? extends HttpMessageConverter<?>> converterType) throws IOException;

    /**
     * 第三步被调用：在请求体已经被转换成参数对象之后被调用
     * Invoked third (and last) after the request body is converted to an Object.
     *
     * 在这里因为已经转换成了对象，到了这一步已经不能修改对应的类型了，但是可以修改对象里面的属性
     * 如果在这里处理，可以通过继承的关系来实现加解密
     */
    Object afterBodyRead(Object body, HttpInputMessage inputMessage, MethodParameter parameter,
            Type targetType, Class<? extends HttpMessageConverter<?>> converterType);

    /**
     * 如果请求体是空时被调用
     * Invoked second (and last) if the body is empty.
     */
    @Nullable
    Object handleEmptyBody(@Nullable Object body, HttpInputMessage inputMessage, MethodParameter parameter,
            Type targetType, Class<? extends HttpMessageConverter<?>> converterType);
}
```

下面详细说一下各个方法的作用

## RequestBodyAdvice#supports 判断是否需要处理请求

通过方法签名可以看出，当返回值为 true 时，需要执行

```java
boolean supports(MethodParameter methodParameter, Type targetType,
            Class<? extends HttpMessageConverter<?>> converterType);
```

示例代码

```java
@Hahahahahahahaha
@PostMapping("abcd")
public Map<String, String> adcd(@RequestBody HahaDTO hahaDTO) {
  Map<String, String> result = new HashMap<>();
  result.put("hello", "world");
  return result;
}
```

通过下图可以看出

- targetType 为要转换的目标类型，也就是 `@RequestBody` 对应的参数
- converterType 为项目使用的 HttpMessageConverter
- methodParameter 为执行过程中要执行的 HandleMethod

在大多数情况我们可以用 methodParameter 来判断是否需要处理该请求，同时我们也可以通过注解的方式来灵活的配置

```java
public boolean supports(MethodParameter methodParameter, Type targetType, 
                        Class<? extends HttpMessageConverter<?>> converterType) {
    // 处理有 Hahahahahahahaha 注解的接口，@Hahahahahahahaha 在上面哦
    return methodParameter.hasMethodAnnotation(Hahahahahahahaha.class);
}
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/a4Fvz13svYkRrKw7uMpjp617NLurdOG5EezR5V1poK1cicr9Z7hBicfVQ9QOXGpWYFf327iakJqwlJd36ZrUxd6Fg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

image-20210324221628299

## RequestBodyAdvice#beforeBodyRead 在请求转换为对象前进行处理

在这个阶段我们可以通过自定义返回 HttpInputMessage 对象，来实现对原请求体数据的修改

```java
HttpInputMessage beforeBodyRead(HttpInputMessage inputMessage, MethodParameter parameter,
            Type targetType, Class<? extends HttpMessageConverter<?>> converterType) throws IOException;
```

假设我们的约定好的请求参数为：

```json
{
    "merchant": "xxxe9293", // 商户号，明文，一般我们这个标识具体商户并找到对应公钥文件
    "data": "加密后的数据", // 这里一般使用对方 RSA 公钥加密
    "sign": "data 原文的数据签名" // 这里使用自己的私钥进行签名
}
```

在我们自定义 HttpInputMessage 之前，先看一下其接口定义，通过接口定义我们可以看出只要实现对应的 getHeaders 和  getBody 即可

```java
public interface HttpMessage {
  HttpHeaders getHeaders();
}

public interface HttpInputMessage extends HttpMessage {
  InputStream getBody() throws IOException;
}
```

这里我们实现一个 嘻嘻 的 HttpInputMessage

```java
public class XixiHttpInputMessage implements HttpInputMessage {

  HttpHeaders headers;
  InputStream body;

  public XixiHttpInputMessage(HttpHeaders headers, InputStream body) {
    this.headers = headers;
    this.body = body;
  }

  @Override
  public InputStream getBody() throws IOException {
    return null;
  }

  @Override
  public HttpHeaders getHeaders() {
    return null;
  }
}
```

这样我们的准备工作就算完成了，下面看看如何实现

```java
  @Override
  public HttpInputMessage beforeBodyRead(HttpInputMessage inputMessage, MethodParameter parameter,
      Type targetType,
      Class<? extends HttpMessageConverter<?>> converterType) throws IOException {

    // 从原 inputMessage 拿到 inputStream  
    InputStream is = inputMessage.getBody();

    // 读取 is, 从中获取到数据 {"merchant": "xxx", data: "xxx", sign: "xxxx"}  
    // 可以使用 JSON 工具类解析出对应 merchant、data、sign，在根据加解密算法进行处理
    // 这里不在详细介绍怎么使用 JSON 工具类和 RSA 的解密验签流程

    // 得到最终的数据，并构建新的 inputMessage，这样就大功告成了
    String result = "解密之后的数据";
    return new XixiHttpInputMessage(inputMessage.getHeaders(),
        new ByteArrayInputStream(result.getBytes()));
  }
```

## RequestBodyAdvice#afterBodyRead  在请求转换为对象后进行处理

```java
Object afterBodyRead(Object body, HttpInputMessage inputMessage, MethodParameter parameter,
            Type targetType, Class<? extends HttpMessageConverter<?>> converterType);
```

在这里 body 其实已经是 `@RequestBody` 对应的参数即 targetType, 同时还要求你返回 targetType 类型，这时在想做强制转换已经很麻烦了，所有这里可以使用继承的方式来实现

```java
@Hahahahahahahaha
@PostMapping("abcd")
public Map<String, String> adcd(@RequestBody HahaDTO hahaDTO) {
  Map<String, String> result = new HashMap<>();
  result.put("hello", "world");
  return result;
}
```

这里请求参数还是

```json
{
    "merchant": "xxxe9293", // 商户号，明文，一般我们这个标识具体商户并找到对应公钥文件
    "data": "加密后的数据", // 这里一般使用对方 RSA 公钥加密
    "sign": "data 原文的数据签名" // 这里使用自己的私钥进行签名
}
```

这里先创建个基类，用于接收加密的请求参数

```java
@Setter
@Getter
@ToString
@NoArgsConstructor
public class MerchantBaseDTO {

  private String merchant;
  private String data;
  private String sign;

}
```

之后我们需要加解密的 `@RequestBody` 对应的参数都继承自他

```java
@Setter
@Getter
@ToString(callSuper = true)
public class HahaDTO extends MerchantBaseDTO {

  private String name;

}
```

通过使用继承来保证自动转换的正确性

**具体处理代码为：**

```java
public Object afterBodyRead(Object body, HttpInputMessage inputMessage, MethodParameter parameter,
      Type targetType,
      Class<? extends HttpMessageConverter<?>> converterType) {
    // 转换成通用的处理
    MerchantBaseDTO merchantBaseDTO = (MerchantBaseDTO) body;

    // 通过 merchant，data，sign 来得到真实的请求数据
    String merchant = merchantBaseDTO.getMerchant();
    String data = merchantBaseDTO.getData();
    String sign = merchantBaseDTO.getSign();
    // 具体加解密验签逻辑 这里并不关心
    String originRequestBody = "通过 merchant，data，sign 来得到真实的请求数据";

    return JSON.parseObject(originRequestBody, targetType);
}
```

> 在上面的解决方案中 beforeBodyRead 和 afterBodyRead 选一个即可
>
> 在上面的解决方案中 beforeBodyRead 和 afterBodyRead 选一个即可
>
> 在上面的解决方案中 beforeBodyRead 和 afterBodyRead 选一个即可

## 具体代码可以为

记住加 `@ControllerAdvice` 注解，在这里例子中 beforeBodyRead 和 afterBodyRead 选一个即可，这里为了参考所有两个都保留了

> 在这里例子中 beforeBodyRead 和 afterBodyRead 选一个即可，
>
> 在这里例子中 beforeBodyRead 和 afterBodyRead 选一个即可，
>
> 在这里例子中 beforeBodyRead 和 afterBodyRead 选一个即可，

```java
@ControllerAdvice
public class RequestAdvice implements RequestBodyAdvice {


  /**
   * Invoked first to determine if this interceptor applies.
   */
  @Override
  public boolean supports(MethodParameter methodParameter, Type targetType,
      Class<? extends HttpMessageConverter<?>> converterType) {
    return true;
  }

  /**
   * Invoked second before the request body is read and converted.
   *
   */
  @Override
  public HttpInputMessage beforeBodyRead(HttpInputMessage inputMessage, MethodParameter parameter,
      Type targetType,
      Class<? extends HttpMessageConverter<?>> converterType) throws IOException {

    InputStream is = inputMessage.getBody();
    //这里灵活的可以支持到多种加解密方式

    String result = "解密之后的数据";
    return new XixiHttpInputMessage(inputMessage.getHeaders(),
        new ByteArrayInputStream(result.getBytes()));
  }


  /**
   * Invoked third (and last) after the request body is converted to an Object.
   */
  @Override
  public Object afterBodyRead(Object body, HttpInputMessage inputMessage, MethodParameter parameter,
      Type targetType,
      Class<? extends HttpMessageConverter<?>> converterType) {
    // 转换成通用的处理
    MerchantBaseDTO merchantBaseDTO = (MerchantBaseDTO) body;

    // 通过 merchant，data，sign 来得到真实的请求数据
    String merchant = merchantBaseDTO.getMerchant();
    String data = merchantBaseDTO.getData();
    String sign = merchantBaseDTO.getSign();

    String originRequestBody = "通过 merchant，data，sign 来得到真实的请求数据";

    return JSON.parseObject(originRequestBody, targetType);
  }

  /**
   * Invoked second (and last) if the body is empty.
   *
   */
  @Override
  public Object handleEmptyBody(Object body, HttpInputMessage inputMessage,
      MethodParameter parameter, Type targetType,
      Class<? extends HttpMessageConverter<?>> converterType) {
    return null;
  }

}
```

## 总结

通过上面的介绍，我们可以通过 RequestBodyAdvice 来修改 请求体 或者修改已经转换完成的对象，来达到修改参数的目的，当然我们也可以通过这个来实现打日志，参数校验等功能