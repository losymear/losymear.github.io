----
title: retrofit使用笔记
date: 2018-10-20 16:38:09
tags:
    - java
toc: true

----
[retrofit](http://square.github.io/retrofit/)基于okhttp，是当下较为流行的Java和Android的HTTP客户端，OOP思路，符合Java哲学。只需要将HTTP请求API封装成一个Java接口，然后就可以使用`retrofitInstace.create(YourApi)`来自动生成这个接口的实例对象，然后用个对象来发送HTTP请求接收返回值。

  <!-- more -->
特点有：
- 支持同步和异步接口
- 支持使用各种Converter来解析返回值
- 适合请求RESTful风格的服务接口
- OOP，可以指定接口被解析成某个类
- 当服务器接口的返回值格式不统一时，就会很难处理（比如返回值是个Json，有一个key是当前日期，这就需要写新的Converter来解析）

## 添加依赖
maven项目添加方法为：
```xml
<dependency>
    <groupId>com.squareup.retrofit2</groupId>
    <artifactId>retrofit</artifactId>
    <version>2.4.0</version>
</dependency>
<!-- 用于解析Json数据 -->
<dependency>
    <groupId>com.squareup.retrofit2</groupId>
    <artifactId>converter-jackson</artifactId>
    <version>2.4.0</version>
</dependency>
```

## 基本使用结构
```java
// 请求url示例：http://httpbin.org/get?key=test
@Log4j2
public class Example {

    public static void main(String[] args) throws Exception {
        // Retrofit类实例
        Retrofit retrofit = new Retrofit.Builder()
                .baseUrl("https://httpbin.org")
                .addConverterFactory(JacksonConverterFactory.create())
                .build();
        // Retrofit自动根据API接口创建实例
        HttpbinApiService service = retrofit.create(HttpbinApiService.class);
        // 创建请求对象，用于同步或异步的发送请求
        Call<HttpGetResult> call = service.testGET("param");
        // 同步方式请求
        Response<HttpGetResult> executed = call.execute();
        HttpGetResult httpGetResult = executed.body();
        log.info("result body {}", httpGetResult);
    }

    // API接口类
    public interface HttpbinApiService {
        @GET("get")
         Call<HttpGetResult> testGET(@Query("param") String param);
    }
}

// 用于接收响应体的类
@JsonIgnoreProperties(ignoreUnknown = true)
@Data
class HttpGetResult {
    public Map<String, String> args;
    public Map<String, String> headers;
}
```

示例中可以看到，使用Retrofit发送请求需要：
1. 定义接收返回数据的类`HttpGetResult`
2. 定义封装了HTTP请求的接口，如示例中的`HttpbinApiService`
3. 创建Retrofit实例，如示例中的`retrofit`
4. 创建实现了HTTP请求接口的实例对象`service`，通过`retrofit.create(myInterface.class)`
5. 调用对象`service`的方法创建一个**发送请求对象**，即这里的`call`。此时并未发请求
6. 使用**发送请求对象**发送请求。示例中的`call.execute()`为同步请求
具体说明如下。

### 创建接收返回数据的类
`retrofit2.Call<T>`是一个泛型，类型参数`<T>`是响应体的类型。总是应该为响应对象创建一个类，并且当能使用具体类型时就尽量使用具体类型，而不是使用`Map`。比如在为`http://httpbin.org/get?param=variable`请求创建返回值类型时，如果你能确定`headers`有固定的字段`Accept`、`Cookie`、`User-Agent`,那么就要为它创建一个具体的类：

```java
@Data
Class Headers(){
    public String Accept;
    public String Cookie;
    public String User-Agent;
}
```

一个具体的类符合OOP，你可以使用`headers.setCookie`或`headers.getCookie`方法来获取/设置cookie，并且你能明确知道这个方法的含义。而一个`Map`只是一种数据结构，`map.set(key,value)`的含义有时不是那么明确。

当然如果返回的字段`headers`是不固定的，那么应该使用`Map`。


### 定义封装HTTP请求的接口
你需要将Http请求抽象成接口，如示例中的`HttpbinApiService`。
接口中的每一个方法，都代表RESTful API的一个接口，如示例中的`@GET("get") Call<HttpGetResult> testGET(@Query("param") String param);`表示向`get`这个路径发送请求GET请求。这个接口方法的具体剖析如下：
- `@GET("get")`表示请求方法是GET，由`@GET`注解定义（除`@GET`之外还有`@POST`、`@DELETE`等注解，对应HTTP方法）；注解的参数定义了请求路径是`"get"`。当然这两者还不能确定请求目标，还缺少`baseURL`，这会在`Retrofit`实例化时设置。
- 返回类型`Call<HttpGetResult>`是一个*发送请求对象*，即示例中的`call`。此时请求还未发出，需要调用该请求来发送同步或者异步请求。`HttpGetResult`是在第一步时创建的类型，用于接收HTTP响应体。
- `(@Query("param") String param)` 接口方法里面的每一个参数都是HTTP请求相关的参数（query参数、请求头、表单字段等），比如示例中的`@Query("param")`表示HTTP的查询参数`param`。需要在调用接口实例的方法时传入。

### 创建`Retrofit`实例并创建接口实例
```java
Retrofit retrofit = new Retrofit.Builder()
        .baseUrl("https://httpbin.org")
        .addConverterFactory(JacksonConverterFactory.create())
   //   .addCallAdapter(adapter)
        .build();

HttpbinApiService service = retrofit.create(HttpbinApiService.class);
```
第一条语句在创建`Retrofit`实例时创建了`baseUrl`，并使用`addConverterFactory`来添加类型转换器。第二条语句用`retrofit.create(HttpbinApiService.class)`创建接口实例。
分成两步的一个好处是按功能解耦复杂的服务接口。比如虽然都是向`myserver.com`发送HTTP请求，但是可能会有多个业务，一个是外卖订单服务一个是打车服务，这样可以定义两个接口`OrderService`和`CallMeCarService`再分别创建实例。
*`//.addCallAdapterFactory(CallAdapter.Factory factory)`*用于添加适配器，用于支持返回类型不是`Call`类型的方法，见[文档](https://square.github.io/retrofit/2.x/retrofit/)，目前没有还没有遇到过使用场景。



### 创建*发送请求对象*并发送请求
`service`对象实现了`HttpbinApiService`，现在就能使用该对象轻松的发送请求接收返回值了。

```java
   Call<HttpGetResult> call = service.testGET("param");
    Response<HttpGetResult> executed = call.execute();
    HttpGetResult httpGetResult = executed.body();
```

`call`对象是调用接口实例对象的方法得到的*发送请求对象*。它支持发起同步请求和发起异步请求，示例是最简单的形式，使用`call.execute()`发送同步请求。此时`executed`保存了返回值，可以用`executed.body()`得到使用转换器转换后的结果，是一个`HttpGetResult`。这个类型正是接口方法中`Call<T>`里的形参！

到此使用`Retrofit`的最简形式已经描述清楚了。


## 同步与异步请求
### 同步地发送请求
即上述示例中的`Response<HttpGetResult> executed = call.execute();`，非常容易使用。

### 异步地发送请求
查阅[`Call` javadoc](https://square.github.io/retrofit/2.x/retrofit/retrofit2/Call.html)，可以看到`call.enqueue(Callback<T> callback)`方法，“Asynchronously send the request and notify callback of its response or if an error occurred talking to the server, creating the request, or processing the response.”而[`Callback`](https://square.github.io/retrofit/2.x/retrofit/retrofit2/Callback.html)接口需要实现`onResponse()`和`onFailure()`方法。

```java
call.enqueue(new Callback<Object>() {
    @Override
    public void onResponse(Call<Object> call, Response<Object> response) {
        System.out.println(response.body());
    }

    @Override
    public void onFailure(Call<Object> call, Throwable throwable) {
        System.out.println(throwable);
    }
});
```

仅此而已，retrofit2本身并未提供并发请求等功能，不过可以在`Callback`实现中使用`Future`，然后用`CompletableFuture.allOf`来并发执行，见[How to make concurrent network requests using OKHTTP?](https://stackoverflow.com/questions/41833314/how-to-make-concurrent-network-requests-using-okhttp)这个so问答。

## 注解
`Retrofit`有8种标注网络请求方法的注解： `@GET`、`@POST`、`PUT`、`DELETE`、`@PATCH`、`@HEAD`、`@OPTIONS`、`@HTTP`（`@HTTP`是对其它7种的扩展，如`@HTTP(method="GET", path="users/{user}", hasBody = false)`），其它的注解主要是用来标注请求参数/请求头/请求体的，具体见[javadoc文档](https://square.github.io/retrofit/2.x/retrofit/overview-tree.html)。

### [`@Url`](https://square.github.io/retrofit/2.x/retrofit/retrofit2/http/Url.html)
手动传入url。此时不需要在`@GET`等方法注解上添加路径。

作用于方法
```Java
    public interface HttpbinAPIService {
     @GET
     Call<Httpbin_GET_Obj> testGETWithUrl(@Url String url, @Query("param") String param);
    }
```
标注`Url`时，`@GET`中不需要设置参数值，不过在调用`service`的方法时需要手动传入。

### [`@Streaming`](https://square.github.io/retrofit/2.x/retrofit/retrofit2/http/Streaming.html)
以流的形式返回结果，而不是将响应体转换成`byte[]`。

### [`@Query`](https://square.github.io/retrofit/2.x/retrofit/retrofit2/http/Query.html)、[`@QueryMap`](https://square.github.io/retrofit/2.x/retrofit/retrofit2/http/QueryMap.html)、[`@QueryName`](https://square.github.io/retrofit/2.x/retrofit/retrofit2/http/QueryName.html)
用于HTTP的query参数。
`@Query`用于单个query参数（并且参数名是限定死的），而`@QueryMap`接收一个Map，可以传入任意个查询query，组装成一个对象。
`@QueryName`会将Query参数加到Url末尾，但是没有值。比如下面示例调用`foo.friends("contains(Bob)", "age(42)")`会得到`/friends?contains(Bob)&age(42)`。
```java
 @GET("/friends")
 Call<ResponseBody> friends(@QueryName String... filters);
```

### [`@MultiPart`](https://square.github.io/retrofit/2.x/retrofit/retrofit2/http/Multipart.html)
标注请求体为multi-part，这适用于有文件上传的场景。请求体的每一部分需要使用`@Part`标注。

```java
@POST("/images/upload")
@Multipart
Call<ResponseBody> fileUpload(@Part("image1") RequestBody image1,@Part("image2") RequestBody image2)
```

### [`@Part`](http://docs.oracle.com/javase/7/docs/api/java/lang/annotation/Annotation.html?is-external=true)和[`@PartMap`](http://docs.oracle.com/javase/7/docs/api/java/lang/annotation/Annotation.html?is-external=true)
`@Part`用于标注作为multi-part请求的一部分。`@PartMap`和`@Part`的关系类似`@QueryMap`和`@Query`的关系。


### [`@Header`](https://square.github.io/retrofit/2.x/retrofit/retrofit2/http/Header.html)、[`@HeadMap`](https://square.github.io/retrofit/2.x/retrofit/retrofit2/http/HeaderMap.html)、[`@Headers`](https://square.github.io/retrofit/2.x/retrofit/retrofit2/http/Headers.html)
`@Header`表示参数是一个请求头。`@HeaderMap`和`@Header`的关系类似`@QueryMap`和`@Query`的关系。

```java
 @GET("/search")
 void list(@HeaderMap Map<String, String> headers);
 @GET("/search2")
 void list2(@Header('Accept') headers);
```

`@Headers`设置了具体的请求头键值，会在请求时带上。比如下面的示例设置了请求的缓存时间：

```java
 @Headers("Cache-Control: max-age=640000")
 @GET("/")
 ...
```
`@Headers`另外一个用途是添加鉴权头部。如果客户端是个合法用户且鉴权信息不会改变，那么直接在方法上加上`@Headers("Authorization: <type> <credentials>")`。

### [`@FormUrlEncoded`](https://square.github.io/retrofit/2.x/retrofit/retrofit2/http/FormUrlEncoded.html)
标注于方法。
表示请求体会作为表单发送，即MIME类型为`application/x-www-form-urlencoded`。具体字段则需要是方法参数标注`@Field(fieldName)`。

```java
 @FormUrlEncoded
    Call<TheResponse> uplaodForm(@Field("name") String name, @Field("sex") int sex);
```

### [`@Field`](https://square.github.io/retrofit/2.x/retrofit/retrofit2/http/Field.html)和[`@FieldMap`](https://square.github.io/retrofit/2.x/retrofit/retrofit2/http/FieldMap.html)
`@Field`表示参数是表单的字段。`@FieldMap`接收一个Map。

### [`@Path`](https://square.github.io/retrofit/2.x/retrofit/retrofit2/http/Body.html)
标注请求URL的某一片段，一般用于用户ID等。Spring MVC有类似功能。

```java
 @GET("/image/{id}/avatar")
 Call<ResponseBody> example(@Path("id") int id);
```

### [`@Body`](https://square.github.io/retrofit/2.x/retrofit/retrofit2/http/Body.html)
使用`@Body`标注一个参数，表示你想要直接控制POST/PUT方法发送的数据，而不是使用请求参数或表单形式的请求体。
比如要[发送纯文本](https://futurestud.io/tutorials/retrofit-2-how-to-send-plain-text-request-body)就需要使用这种方式。
```java
public interface ScalarService {
    @POST("path")
    Call<String> getStringScalar(@Body String body);
}
```

另外如果POST需要发送有较多字段的JSON数据，可以先将请求数据封装为一个DTO类，再用`@Body`标注请求参数来发送数据。retrofit会将你定义的DTO类转换为raw json，下面的案例即是这种方式，也可以查看[How to POST raw whole JSON in the body of a Retrofit request?
](https://stackoverflow.com/questions/21398598/how-to-post-raw-whole-json-in-the-body-of-a-retrofit-request)这个so问答。

## 项目中的一个案例
微信小程序转生成二维码的接口很神奇，如果请求出错会返回JSON（包含错误信息、错误码等），而请求成功返回的是二进制数据流。
具体代码就不给了，只给出处理的核心代码：

```java
public interface MiniProgramService{
    @POST("wxa/getwxacode")
    public Call<ResponseBody> getwxacode(@Body XacodeType xacodeType, @Query("access_token") String access_token);
}
```

`XacodeType`是小程序二维码接口需要的请求体数据，包括返回图片的长宽、颜色、是否透明等。 因为返回图片成功和失败时的格式不统一，无法统一处理，只能使用`okhttp3.ResponseBody`作为返回数据格式。然后给定一个`ErrorType`对应错误数据的类型。如果返回的是个错误信息JSON，可以使用`response.body().string()`得到JSON字符串，再用Jackson将数据转换成`ErrorType`。
转换失败说明返回的不是`ErrorType`类型的JSON，而是图片的二进制数据，那么就该如下处理（返回的图片数据要转换成`byte[]`）：使用`byte[] bytedata = responseData.body().source().readByteArray();`获取二进制数据，然后通过` String imageData = org.apache.commons.codec.binary.Base64.encodeBase64String(bytedata);`转换成base64编码的字符串。最终前端使用base64显示图片`src = "data:image/png;base64,${imageData}"`。



## 其它HTTP客户端框架
*使用`Retrofit2`已经足够，以下内容主要用于个人记录方便以后回忆，只是一些碎片记录，请跳过*

### Spring的`RestTemplate`
#### `UriComponentsBuilder`
允许通过在URI模板中使用变量：

```java
UriComponents uriComponents = UriComponentsBuilder
        .fromUriString("http://example.com/hotels/{hotel}")
        .queryParam("q", "{q}")
        .encode()
        .build();

URI uri = uriComponents.expand("Westin", "123").toUri();
```
可以简化成：
```java
URI uri = UriComponentsBuilder
        .fromUriString("http://example.com/hotels/{hotel}")
        .queryParam("q", "{q}")
        .build("Westin", "123");
```

#### `UriBuilder`
`UriComponentsBuilder`类实现了`UriBuilder`接口。 一个`UriBuilder`可以通过`UriBuilderFactory`创建。
`UriBuilder`和`UriBuilderFactory`提供了一种可插拔的机制来从URI模板中创建URI，需要设置一些共享配置如base url、encoding等。

`RestTemplate`和`WebClient`可以使用`UriBuilderFactory`来进行URI的准备工作。
`RestTemplate`示例：
```java
// import org.springframework.web.util.DefaultUriBuilderFactory.EncodingMode;

String baseUrl = "http://example.org";
DefaultUriBuilderFactory factory = new DefaultUriBuilderFactory(baseUrl);
factory.setEncodingMode(EncodingMode.TEMPLATE_AND_VARIABLES);

RestTemplate restTemplate = new RestTemplate();
restTemplate.setUriTemplateHandler(factory);
```

直接使用：
```java
String baseUrl = "http://example.com";
DefaultUriBuilderFactory uriBuilderFactory = new DefaultUriBuilderFactory(baseUrl);

URI uri = uriBuilderFactory.uriString("/hotels/{hotel}")
        .queryParam("q", "{q}")
        .build("Westin", "123");
```

#### URI Encoding

## read more
- [introduction](https://square.github.io/retrofit/)
- [Retrofit-Tutorials](https://github.com/square/retrofit/wiki/Retrofit-Tutorials)
- [why use Retrofit when we have OkHttp](https://stackoverflow.com/questions/39183294/why-use-retrofit-when-we-have-okhttp)