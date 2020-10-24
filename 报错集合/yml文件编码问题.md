# java.lang.IllegalStateException: Failed to load property source from location 'classpath:/application.yml'

前一天代码正常运行，第二天起来提示yml文件中UTF乱码，于是将编码改成GBK，然而我的项目本身是UTF8的。

然后再运行，就出现了如题BUG。

算是一个借鉴，出现这种错误