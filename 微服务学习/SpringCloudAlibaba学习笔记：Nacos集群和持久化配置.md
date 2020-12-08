# Nacos集群和持久化配置

## Nacos集群部署说明

> https://nacos.io/zh-cn/docs/cluster-mode-quick-start.html

Nacos的集群部署架构图如下：

![1561258986171-4ddec33c-a632-4ec3-bfff-7ef4ffc33fb9](img/SpringCloudAlibaba%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%9ANacos%E9%9B%86%E7%BE%A4%E5%92%8C%E6%8C%81%E4%B9%85%E5%8C%96%E9%85%8D%E7%BD%AE/1561258986171-4ddec33c-a632-4ec3-bfff-7ef4ffc33fb9.jpeg)

此处的VIP是虚拟映射IP，可以由Nginx实现。

默认nacos使用嵌入式的数据库实现数据的存储，所以，如果启动多个默认配置下的Nacos节点，数据存储是存在一致性问题的。为了解决这个问题，Nacos采用集中式存储的方式来支持集群化部署，目前只支持MySQL的存储。

## Nacos的持久化配置

默认nacos使用嵌入式的数据库derby实现数据的存储，Nacos也支持mysql实现持久化配置。

![2020062211513644](img/SpringCloudAlibaba%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%9ANacos%E9%9B%86%E7%BE%A4%E5%92%8C%E6%8C%81%E4%B9%85%E5%8C%96%E9%85%8D%E7%BD%AE/2020062211513644.png)

