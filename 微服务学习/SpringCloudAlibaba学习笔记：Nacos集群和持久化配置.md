# Nacos集群和持久化配置

## Nacos集群部署说明

> https://nacos.io/zh-cn/docs/cluster-mode-quick-start.html

Nacos的集群部署架构图如下：

![1561258986171-4ddec33c-a632-4ec3-bfff-7ef4ffc33fb9](img/SpringCloudAlibaba%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%9ANacos%E9%9B%86%E7%BE%A4%E5%92%8C%E6%8C%81%E4%B9%85%E5%8C%96%E9%85%8D%E7%BD%AE/1561258986171-4ddec33c-a632-4ec3-bfff-7ef4ffc33fb9.jpeg)

此处的VIP是虚拟映射IP，可以由Nginx实现。

默认nacos使用嵌入式的数据库实现数据的存储，所以，如果启动多个默认配置下的Nacos节点，数据存储是存在一致性问题的。为了解决这个问题，Nacos采用集中式存储的方式来支持集群化部署，目前只支持MySQL的存储。

## Nacos的部署模式

> https://nacos.io/zh-cn/docs/deployment.html

Nacos支持三种部署模式：

1. 单机模式，用于测试和单机试用，我们之前使用的就是单机模式启动。
2. 集群模式，用于生产环境，确保高可用。
3. 多集群模式，用于多数据中心场景。

## 单机模式支持mysql持久化

在单机模式下，0.7版本之前，默认nacos使用嵌入式的数据库derby实现数据的存储，不方便观察数据存储的基本情况。

0.7版本增加了支持mysql数据源能力，具体的操作步骤：

一、安装数据库，版本要求：5.6.5+。

二、初始化mysql数据库，数据库初始化文件：`conf/nacos-mysql.sql`。执行这个脚本之前，需要按照脚本中的提示，创建指定的数据库。

```sql
/******************************************/
/*   数据库全名 = nacos_config   */
/******************************************/
```

三、修改`conf/application.properties`文件，增加支持mysql数据源配置（目前只支持mysql），添加mysql数据源的url、用户名和密码。只要把properties中的注释打开就可以了。

![image-20201216000659355](img/SpringCloudAlibaba%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%9ANacos%E9%9B%86%E7%BE%A4%E5%92%8C%E6%8C%81%E4%B9%85%E5%8C%96%E9%85%8D%E7%BD%AE/image-20201216000659355.png)

再以单机模式启动nacos，nacos所有写嵌入式数据库的数据都写到了mysql。

测试一下，新建一个配置，在`config_info`就将存入一条记录。