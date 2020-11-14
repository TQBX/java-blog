[toc]

# 利用SpirngBoot实现文件上传功能

## 零、本篇要点

- 介绍SpringBoot对文件上传的自动配置。
- 介绍MultipartFile接口。
- 介绍SpringBoot+Thymeleaf文件上传demo的整合。
- 介绍对文件类型，文件名长度等判断方法。

## 一、SpringBoot对文件处理相关自动配置

自动配置是SpringBoot为我们提供的便利之一，开发者可以在不作任何配置的情况下，使用SpringBoot提供的默认设置，如处理文件需要的MultipartResolver。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ Servlet.class, StandardServletMultipartResolver.class, MultipartConfigElement.class })
@ConditionalOnProperty(prefix = "spring.servlet.multipart", name = "enabled", matchIfMissing = true)
@ConditionalOnWebApplication(type = Type.SERVLET)
@EnableConfigurationProperties(MultipartProperties.class)
public class MultipartAutoConfiguration {

	private final MultipartProperties multipartProperties;

	public MultipartAutoConfiguration(MultipartProperties multipartProperties) {
		this.multipartProperties = multipartProperties;
	}

	@Bean
	@ConditionalOnMissingBean({ MultipartConfigElement.class, CommonsMultipartResolver.class })
	public MultipartConfigElement multipartConfigElement() {
		return this.multipartProperties.createMultipartConfig();
	}

	@Bean(name = DispatcherServlet.MULTIPART_RESOLVER_BEAN_NAME)
	@ConditionalOnMissingBean(MultipartResolver.class)
	public StandardServletMultipartResolver multipartResolver() {
		StandardServletMultipartResolver multipartResolver = new StandardServletMultipartResolver();
		multipartResolver.setResolveLazily(this.multipartProperties.isResolveLazily());
		return multipartResolver;
	}

}
```

- Spring3.1之后支持`StandardServletMultipartResolver`，且默认使用`StandardServletMultipartResolver`，它的优点在于：使用Servlet所提供的功能支持，不需要依赖任何其他的项目。
- 想要自动配置生效，需要配置`spring.servlet.multipart.enabled=true`，当然这个配置默认就是true。
- 相关的配置设置在`MultipartProperties`中，其中字段就是对应的属性设置，经典字段有：
  - `enabled`：是否开启文件上传自动配置，默认开启。
  - `location`：上传文件的临时目录。
  - `maxFileSize`：最大文件大小，以字节为单位，默认为1M。
  - `maxRequestSize`：整个请求的最大容量，默认为10M。
  - `fileSizeThreshold`：文件大小达到该阈值，将写入临时目录，默认为0，即所有文件都会直接写入磁盘临时文件中。
  - `resolveLazily`：是否惰性处理请求，默认为false。
- 我们也可以自定义处理的细节，需要实现MultipartResolver接口。

## 二、处理上传文件MultipartFile接口

SpringBoot为我们提供了MultipartFile强大接口，让我们能够获取上传文件的详细信息，如原始文件名，内容类型等等，接口内容如下：

```java
public interface MultipartFile extends InputStreamSource {
    String getName(); //获取参数名
    @Nullable
    String getOriginalFilename();//原始的文件名
    @Nullable
    String getContentType();//内容类型
    boolean isEmpty();
    long getSize(); //大小
    byte[] getBytes() throws IOException;// 获取字节数组
    InputStream getInputStream() throws IOException;//以流方式进行读取
    default Resource getResource() {
        return new MultipartFileResource(this);
    }
    // 将上传的文件写入文件系统
    void transferTo(File var1) throws IOException, IllegalStateException;
	// 写入指定path
    default void transferTo(Path dest) throws IOException, IllegalStateException {
        FileCopyUtils.copy(this.getInputStream(), Files.newOutputStream(dest));
    }
}
```

## 三、SpringBoot+Thymeleaf整合demo

### 1、编写控制器

```java
/**
 * 文件上传
 *
 * @author Summerday
 */
@Controller
public class FileUploadController {

    private static final String UPLOADED_FOLDER = System.getProperty("user.dir");

    @GetMapping("/")
    public String index() {
        return "file";
    }

    @PostMapping("/upload")
    public String singleFileUpload(@RequestParam("file") MultipartFile file,
                                   RedirectAttributes redirectAttributes) throws IOException {

        if (file.isEmpty()) {
            redirectAttributes.addFlashAttribute("msg", "文件为空,请选择你的文件上传");
            return "redirect:uploadStatus";

        }
        saveFile(file);
        redirectAttributes.addFlashAttribute("msg", "上传文件" + file.getOriginalFilename() + "成功");
        redirectAttributes.addFlashAttribute("url", "/upload/" + file.getOriginalFilename());
        return "redirect:uploadStatus";
    }

    private void saveFile(MultipartFile file) throws IOException {
        Path path = Paths.get(UPLOADED_FOLDER + "/" + file.getOriginalFilename());
        file.transferTo(path);
    }

    @GetMapping("/uploadStatus")
    public String uploadStatus() {
        return "uploadStatus";
    }
}
```

### 2、编写页面file.html

```html
<html xmlns:th="http://www.thymeleaf.org">
    <!--suppress ALL-->
<html lang="en">
    <head>
        <meta charset="UTF-8">
        <title>文件上传界面</title>
    </head>
    <body>
        <div>
            <form method="POST" enctype="multipart/form-data" action="/upload">
                <table>
                    <tr><td><input type="file" name="file" /></td></tr>
                    <tr><td></td><td><input type="submit" value="上传" /></td></tr>
                </table>
            </form>

        </div>
    </body>
</html>
```

### 3、编写页面uploadStatus.html

```html
<!--suppress ALL-->
<html xmlns:th="http://www.thymeleaf.org">

<html lang="en">
    <head>
        <meta charset="UTF-8">
        <title>文件上传界面</title>
    </head>
    <body>
        <div th:if="${msg}">
            <h2 th:text="${msg}"/>
        </div>
        <div >
            <img src="" th:src="${url}" alt="">
        </div>
    </body>
</html>
```

### 4、编写配置

```properties
server.port=8081
spring.servlet.multipart.max-file-size=10MB
spring.servlet.multipart.max-request-size=10MB
```

### 5、配置虚拟路径映射

这一步是非常重要的，我们将文件上传到服务器上时，我们需要将我们的请求路径和服务器上的路径进行对应，不然很有可能文件上传成功，但访问失败：

```java
@Configuration
public class MvcConfig implements WebMvcConfigurer {

    private static final String UPLOADED_FOLDER = System.getProperty("user.dir");

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/upload/**")
                .addResourceLocations("file:///" + UPLOADED_FOLDER + "/");
    }
}
```

对应关系需要自己去定义，如果访问失败，可以试着打印以下路径，看看是否缺失了路径分隔符。

> 注意：如果addResourceHandler不要写成处理/**，这样会拦截掉其他的请求

### 6、测试页面

执行`mvn spring-boot:run`，启动程序，访问`http://localhost:8081/`，选择文件，点击上传按钮，我们的项目目录下出现了mongo.jpg，并且页面也成功显示：

![image-20201113140155425](img/SpringBoot%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0/image-20201113140155425.png)

## 四、SpringBoot的Restful风格，返回url

```java
/**
 * 文件上传
 *
 * @author Summerday
 */
@RestController
public class FileUploadRestController {

    /**
     * 文件名长度
     */
    private static final int DEFAULT_FILE_NAME_LENGTH = 100;

    /**
     * 允许的文件类型
     */
    private static final String[] ALLOWED_EXTENSIONS = {
            "jpg", "img", "png", "gif"
    };

    /**
     * 项目路径
     */
    private static final String UPLOADED_FOLDER = System.getProperty("user.dir");

    @PostMapping("/restUpload")
    public Map<String,Object> singleFileUpload(@RequestParam("file") MultipartFile file) throws Exception {

        if (file.isEmpty()) {
            throw new Exception("文件为空!");
        }
        String filename = upload(file);
        String url = "/upload/" + filename;
        Map<String,Object> map = new HashMap<>(2);
        map.put("msg","上传成功");
        map.put("url",url);
        return map;
    }


    /**
     * 上传方法
     */
    private String upload(MultipartFile file) throws Exception {
        int len = file.getOriginalFilename().length();
        if (len > DEFAULT_FILE_NAME_LENGTH) {
            throw new Exception("文件名超出限制!");
        }
        String extension = getExtension(file);
        if(!isValidExtension(extension)){
            throw new Exception("文件格式不正确");
        }
        // 自定义文件名
        String filename = getPathName(file);
        // 获取file对象
        File desc = getFile(filename);
        // 写入file
        file.transferTo(desc);
        return filename;
    }

    /**
     * 获取file对象
     */
    private File getFile(String filename) throws IOException {
        File file = new File(UPLOADED_FOLDER + "/" + filename);
        if(!file.getParentFile().exists()){
            file.getParentFile().mkdirs();
        }
        if(!file.exists()){
            file.createNewFile();
        }
        return file;
    }

    /**
     * 验证文件类型是否正确
     */
    private boolean isValidExtension(String extension) {
        for (String allowedExtension : ALLOWED_EXTENSIONS) {
            if(extension.equalsIgnoreCase(allowedExtension)){
                return true;
            }
        }
        return false;
    }

    /**
     * 此处自定义文件名,uuid + extension
     */
    private String getPathName(MultipartFile file) {
        String extension = getExtension(file);
        return UUID.randomUUID().toString() + "." + extension;
    }

    /**
     * 获取扩展名
     */
    private String getExtension(MultipartFile file) {
        String originalFilename = file.getOriginalFilename();
        return originalFilename.substring(originalFilename.lastIndexOf('.') + 1);
    }
}
```

## 五、源码下载

本文内容均为对优秀博客及官方文档总结而得，原文地址均已在文中参考阅读处标注。最后，文中的代码样例已经全部上传至Gitee：[https://gitee.com/tqbx/springboot-samples-learn](https://gitee.com/tqbx/springboot-samples-learn)，另有其他SpringBoot的整合哦。

## 六、参考阅读

- [官方文档：SpringWebMVC#DispatcherServlet#Multipart Resolver](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-multipart)

- [Spring Boot file upload example](https://mkyong.com/spring-boot/spring-boot-file-upload-example/)