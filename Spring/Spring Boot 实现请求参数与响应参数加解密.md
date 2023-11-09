# Spring Boot 实现请求参数与响应参数加解密

## 一、需求

> 只针对@RequestBody @ResponseBody两个注解起作用
>

期望在**request**请求进入controller前做是否**加密**验证，**response**在返回前做是否**解密**验证

## 二、设计

- 添加自定义注解 `EncryptResponse`加密注解，`DecryptRequest`解密注解（使用范围类与方法上）
- 添加一个加解密注解的判定类。
- 继承`RequestBodyAdvice`重写`beforeBodyWrite`方法结合判定类与外部配置确认调用是否需要加解密
  `ResponseBodyAdvice`重写`beforeBodyRead`方法结合判定类与外部配置确认调用是否需要加解密

**RequestBodyAdvice** 与 **ResponseBodyAdvice**

![image-20221021104754617](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/202210211048470.png)

```java
public interface RequestBodyAdvice {

	// 判断是否拦截（可以更精细化地进行判断是否拦截）
	boolean supports(MethodParameter methodParameter, Type targetType,
			Class<? extends HttpMessageConverter<?>> converterType);

	// 进行请求前的拦截处理
	HttpInputMessage beforeBodyRead(HttpInputMessage inputMessage, MethodParameter parameter,
			Type targetType, Class<? extends HttpMessageConverter<?>> converterType) throws IOException;

	// 进行请求后的拦截处理
	Object afterBodyRead(Object body, HttpInputMessage inputMessage, MethodParameter parameter,
			Type targetType, Class<? extends HttpMessageConverter<?>> converterType);

	// 对空请求体的处理
	@Nullable
	Object handleEmptyBody(@Nullable Object body, HttpInputMessage inputMessage, MethodParameter parameter,
			Type targetType, Class<? extends HttpMessageConverter<?>> converterType);
}
```

![image-20221021105431561](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/202210211054655.png)

```java
public interface ResponseBodyAdvice<T> {

	// 判断是否拦截（可以更精细化地进行判断是否拦截）
	boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType);

	// 进行响应前的拦截处理
	@Nullable
	T beforeBodyWrite(@Nullable T body, MethodParameter returnType, MediaType selectedContentType,
			Class<? extends HttpMessageConverter<?>> selectedConverterType,
			ServerHttpRequest request, ServerHttpResponse response);

}
```

## 三、实现

![时序图](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/202210211601101.png)

### 1.自定义加解密的注解

```java
/**
 * @author wanxianbo
 * @description 加了此注解的接口(true)将进行数据解密操作(post的body) 可以放在类上，可以放在方法上
 * @date 创建于 2022/10/21
 */
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface DecryptRequest {
    /**
     * 是否对body进行解密
     */
    boolean value() default true;
}
```

```java
/**
 * @author wanxianbo
 * @description 加了此注解的接口(true)将进行数据加密操作可以放在类上，可以放在方法上
 * @date 创建于 2022/10/21
 */
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface EncryptResponse {
    /**
     * 是否对body进行解密
     */
    boolean value() default true;
}
```

### 2.实现RequestBodyAdvice、ResponseBodyAdvice 接口

```java
/**
 * @author wanxianbo
 * @description 请求数据接收处理类 <br>
 * 对加了@Decrypt的方法的数据进行解密操作<br>
 * 只对 @RequestBody 参数有效
 * @date 创建于 2022/10/21
 */
@ControllerAdvice(basePackages = "com.xkcoding")
public class DecryptRequestBodyAdvice implements RequestBodyAdvice {

    @Value("${spring.crypto.request.decrypt.charset:UTF-8}")
    private String charset = "UTF-8";

    private static final byte[] KEYS = "12345678abcdefgh".getBytes(StandardCharsets.UTF_8);

    @Override
    public boolean supports(MethodParameter methodParameter, Type targetType, Class<? extends HttpMessageConverter<?>> converterType) {
        return true;
    }

    @Override
    public HttpInputMessage beforeBodyRead(HttpInputMessage inputMessage, MethodParameter parameter, Type targetType, Class<? extends HttpMessageConverter<?>> converterType) throws IOException {
        return NeedCrypto.needDecrypt(parameter) ? new HttpInputMessage() {
            @Override
            public InputStream getBody() throws IOException {
                AES aes = SecureUtil.aes(KEYS);
                String content = StreamUtils.copyToString(inputMessage.getBody(), Charset.forName(charset));
                String decryptBody = aes.decryptStr(content);
                return new ByteArrayInputStream(decryptBody.getBytes(Charset.forName(charset)));
            }

            @Override
            public HttpHeaders getHeaders() {
                return inputMessage.getHeaders();
            }
        } : inputMessage;
    }

    @Override
    public Object afterBodyRead(Object body, HttpInputMessage inputMessage, MethodParameter parameter, Type targetType, Class<? extends HttpMessageConverter<?>> converterType) {
        return body;
    }

    @Override
    public Object handleEmptyBody(Object body, HttpInputMessage inputMessage, MethodParameter parameter, Type targetType, Class<? extends HttpMessageConverter<?>> converterType) {
        return body;
    }
}
```

```java
/**
 * @author wanxianbo
 * @description 对加了@EncryptResponse的方法的数据进行加密操作
 * @date 创建于 2022/10/21
 */
@ControllerAdvice(basePackages = "com.xkcoding")
public class EncryptResponseBodyAdvice implements ResponseBodyAdvice<Object> {

    private static final Logger log = LoggerFactory.getLogger(EncryptResponseBodyAdvice.class);

    private static final byte[] KEYS = "12345678abcdefgh".getBytes(StandardCharsets.UTF_8);

    @Autowired
    private ObjectMapper objectMapper;

    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        return NeedCrypto.needEncrypt(returnType);
    }

    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType, Class<? extends HttpMessageConverter<?>> selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) {
        // 对body进行封装处理
        if (StringUtils.equals(returnType.getMethod().getReturnType().getName(), Void.TYPE.getName())) {
            return extract(ResponseEntity.ok(body), returnType);
        } else
        {
            if (body instanceof ResponseEntity) {
                return extract(body, returnType);
            }
        }

        return extract(ResponseEntity.ok(body), returnType);
    }

    private Object extract(Object body, MethodParameter methodParameter) {
        try {
            String content = objectMapper.writeValueAsString(body);
            AES aes = SecureUtil.aes(KEYS);
            return aes.encryptHex(content);
        } catch (Exception e) {
            log.error("对方法method :【" + methodParameter.getMethod().getName() + "】返回数据进行解密出现异常：", e);
            throw new RuntimeException("加密 失败");
        }
    }
}
```

```java
/**
 * @author wanxianbo
 * @description 判断是否需要加解密
 * @date 创建于 2022/10/21
 */
public class NeedCrypto {
    private NeedCrypto() {

    }

    /**
     * 是否需要参数加密
     * 1.类上标注或者方法上标注,并且都为true
     * 2.有一个标注为false就不需要解密
     *
     * @param methodParameter 参数
     * @return boolean
     */
    public static boolean needEncrypt(MethodParameter methodParameter) {
        boolean encrypt = false;
        boolean classPresentAnno = methodParameter.getContainingClass().isAnnotationPresent(EncryptResponse.class);
        boolean methodPresentAnno = Objects.requireNonNull(methodParameter.getMethod()).isAnnotationPresent(EncryptResponse.class);

        if (classPresentAnno) {
            // 类上标注是否需要解密
            encrypt = methodParameter.getContainingClass().getAnnotation(EncryptResponse.class).value();
            // 类上不加密，所有的都不加密
            return encrypt;
        }
        if (methodPresentAnno) {
            // 方法上标注是否需要解密
            encrypt = Objects.requireNonNull(methodParameter.getMethod()).getAnnotation(EncryptResponse.class).value();
        }
        return encrypt;
    }

    /**
     * 是否需要参数解密
     * 1.类上标注或者方法上标注,并且都为true
     * 2.有一个标注为false就不需要解密
     *
     * @param methodParameter 参数
     * @return boolean
     */
    public static boolean needDecrypt(MethodParameter methodParameter) {
        boolean encrypt = false;
        boolean classPresentAnno = methodParameter.getContainingClass().isAnnotationPresent(DecryptRequest.class);
        boolean methodPresentAnno = Objects.requireNonNull(methodParameter.getMethod()).isAnnotationPresent(DecryptRequest.class);

        if (classPresentAnno) {
            // 类上标注是否需要解密
            encrypt = methodParameter.getContainingClass().getAnnotation(DecryptRequest.class).value();
            // 类上不加密，所有的都不加密
            return encrypt;
        }
        if (methodPresentAnno) {
            // 方法上标注是否需要解密
            encrypt = Objects.requireNonNull(methodParameter.getMethod()).getAnnotation(DecryptRequest.class).value();
        }
        return encrypt;
    }
}
```

