### SpringBoot处理全局同意异常

在后端发生异常或者是请求出错时，前端通常显示如下

```txt
Whitelabel Error Page
This application has no explicit mapping for /error, so you are seeing this as a fallback.

Fri Jun 07 15:38:07 CST 2019
There was an unexpected error (type=Not Found, status=404).
No message available
```

 对于用户来说非常不友好。

本文主要讲解如何在SpringBoot应用中使用统一异常处理。

> 实现方式

第一种：使用@ControllerAdvice和@ExceptionHandler注解

第二种: 使用ErrorController类来实现。

#### 第一种：使用@ControllerAdvice和@ExceptionHandler注解

```java
@ControllerAdvice
@Slf4j
public class GlobalExceptionHandler{
    @ResponseBody
    @ExceptionHandler(CustomerException.class)
    public BaseResult handlerCustomerException(CustomerException ex){
        log.info("GlobalExceptionHandler...");
		log.info("错误代码："  + response.getStatus());
		BaseResult result = new BaseResult(ex.getExceptionEumns().getCode,"GlobalExceptionHandler:"+ex.getExceptionEumns().getMessage());
        return result;
    }
}
```

注解@ControllerAdvice表示这是一个控制器增强类，当控制器发生异常且符合类中定义的拦截异常类，将会被拦截。

可以定义拦截的控制器所在的包路径

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface ControllerAdvice {
    @AliasFor("basePackages")
    String[] value() default {};

    @AliasFor("value")
    String[] basePackages() default {};

    Class<?>[] basePackageClasses() default {};

    Class<?>[] assignableTypes() default {};

    Class<? extends Annotation>[] annotations() default {};
}
```

注解ExceptionHandler定义拦截的异常类

```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ExceptionHandler {
    Class<? extends Throwable>[] value() default {};
}
```

#### 第二种: 使用ErrorController类来实现。

系统默认的错误处理类为BasicErrorController，将会显示如上的错误页面。

这里编写一个自己的错误处理类，上面默认的处理类将不会起作用。

getErrorPath()返回的路径服务器将会重定向到该路径对应的处理类，本例中为error方法。

```java
@Slf4j
@RestController
public class HttpErrorController implements ErrorController {

    private final static String ERROR_PATH = "/error";

    @ResponseBody
    @RequestMapping(path  = ERROR_PATH )
    public BaseResult error(HttpServletRequest request, HttpServletResponse response){
        log.info("访问/error" + "  错误代码："  + response.getStatus());
        BaseResult result = new WebResult(WebResult.RESULT_FAIL,"HttpErrorController error:"+response.getStatus());return result;
    }
    @Override
    public String getErrorPath() {
        return ERROR_PATH;
    }
}
```

#### 测试

以上定义了一个统一的返回类BaseResult，方便前端进行处理。

```java
@Data
@NoArgsConstructor
public class BaseResult implements Serializable {

    private static final long serialVersionUID = 1L;

    public static final int RESULT_FAIL = 0;
    public static final int RESULT_SUCCESS = 1;

    //返回代码
    private Integer  code;

    //返回消息
    private String message;

    //返回对象
    private  Object data;

    public BaseResult(Integer code, String message) {
        this.code = code;
        this.message = message;
    }

    public BaseResult(Integer code, String message, Object object) {
        this.code = code;
        this.message = message;
        this.data = object;
    }
}
```

##### 编写一个测试控制器

```java
@Slf4j
@RestController
@RequestMapping("/user")
public class TestController {

    @RequestMapping("/info1")
    public String test(){
      log.info("/user/info1");
      throw new CustomerException("TestController have exception");
    }
}
```

1.发出一个错误的请求,也就是没有对应的处理类。

从返回可以看到是由HttpErrorController类处理

```json
{"code":0,"message":"HttpErrorController error:404","data":null}
```

2.发出一个正常的请求(TestController的test()处理)，处理类中抛出空异样

从返回中可以看出是由GlobalExceptionHandler类处理

```json
{"code":0,"message":"request error:200","data":"GlobalExceptionHandler:TestController have exception"}
```

##### 区别

1.注解@ControllerAdvice方式只能处理控制器抛出的异常。此时请求已经进入控制器中。

2.类ErrorController方式可以处理所有的异常，包括未进入控制器的错误，比如404,401等错误

3.如果应用中两者共同存在，则@ControllerAdvice方式处理控制器抛出的异常，类ErrorController方式未进入控制器的异常。

4.@ControllerAdvice方式可以定义多个拦截方法，拦截不同的异常类，并且可以获取抛出的异常信息，自由度更大。