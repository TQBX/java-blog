### 开发工具

IDEA2020.3

### 异常情况如下

已知项目的默认编码方式为UTF-8，而yml的默认编码方式为GBK。

在yml中配置`com.hyh.name`值为`天乔巴夏`。

通过`@Value("${com.hyh.name}")`读取，产生中文乱码的问题。

### 问题解决

File -> Settings -> Editor -> File Encodings

![image-20201109122701246](img/%E8%A7%A3%E5%86%B3yml%E4%B8%AD%E9%85%8D%E7%BD%AE%E4%B8%AD%E6%96%87%E8%BE%93%E5%87%BA%E4%B9%B1%E7%A0%81%E9%97%AE%E9%A2%98/image-20201109122701246.png)

