## Spring Boot的 @Value 和@ConfigurationProperties 的 使用

### 一、@Value

Spring EL表达式语言，支持在XML和注解中表达式，类是于JSP的EL表达式语言。

在Spring开发中经常涉及调用各种资源的情况，包含普通文件、网址、配置文件、系统环境变量等，我们可以使用Spring的表达式语言实现资源的注入。

Spring主要在注解@Value的参数中使用表达式。

- 注入普通字符串
- 注入操作系统属性
- 注入表达式运算结果
- 注入其他Bean的属性
- 注入文件内容
- 注入网址内容
- 注入属性文件（注意：用的是$符号）

```java
@Component
public class ELConfig {  
    @Value("注入普通字符串")// 注入普通字符串  
    private String normal;  
      
    @Value("#{systemProperties['os.name']}")// 注入操作系统属性  
    private String osName;  
      
    @Value("#{T(java.lang.Math).random() * 100.0 }")// 注入表达式结果  
    private double randomNumber;   
 
    @Value("#{payOrderQueryController.payCenterFacade}")// 注入其他Bean属性
    private IPayCenterFacade fromAnother;
      
    @Value("classpath:test.txt")// 注入文件资源  
    private Resource testFile;  
      
    @Value("https://www.baidu.com")// 注入网址资源  
    private Resource testUrl;  
  
    @Value("${book.name}")// 注入配置文件【注意是$符号】  
    private String bookName;  
      
    @Autowired// Properties可以从Environment获得  
    private Environment environment;  
  
    @Override  
    public String toString() {  
        try {  
            return "ELConfig [normal=" + normal   
                    + ", osName=" + osName   //os.name,如Windows 8.1
                    + ", randomNumber=" + randomNumber   //值如97.53293482705482
                    + ", fromAnother=" + fromAnother   //别的bean的成员属性
                    + ", testFile=" + IOUtils.toString(testFile.getInputStream())   //输出文件里的内容
                    + ", testUrl=" + IOUtils.toString(testUrl.getInputStream())   //输出网页的html
                    + ", bookName=" + bookName  //配置的值
                    + ", environment=" + environment.getProperty("book.name") + "]";  
        } catch (IOException e) {  
            e.printStackTrace();  
            return null;  
        }  
    }  
      
}
```



### 二、Spring Boot 的 @ConfigurationProperties

先看下面的@Value注解：

```
    @Value("${book.name}")
    private String bookName;
    @Value("${book.author}")
    private String bookAuthor;
```

 上面这种使用@Value注入每个配置在实际项目中会显得格外麻烦，因为我们的配置通常会是许多个，就要使用@Value注入很多次。

Spring Boot提供了基于类型安全的配置方式，通过@ConfigurationProperties 将properties属性和一个Bean关联，从而实现类型安全的配置。

```
@Component
@ConfigurationProperties(prefix = "book")
public class Book {

    private String name;
    private String author;
    private int age;
    
    //get.. set..
}
```

 

@ConfigurationProperties有两个属性

- prefix：指定properties的配置的前缀
- locations：指定properties文件的位置