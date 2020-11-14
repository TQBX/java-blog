https://pkg.jenkins.io/redhat-stable/

```shell
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
```

```shell
yum install jenkins
```

https://www.jenkins.io/doc/book/installing/linux/

启动jenkins服务，查看状态

```shell
sudo systemctl start jenkins
sudo systemctl status jenkins
```

找一下jenkins在哪

```shell
whereis jenkins
/usr/lib/jenkins
```

修改jenkins启动的端口号为30005

```shell
java -jar jenkins.war --httpPort=30005
```

开启30005端口号

```shell
firewall-cmd --list-ports
firewall-cmd --zone=public --add-port=30005/tcp --permanent
firewall-cmd --reload
```



关闭和重启Jenkins

- 在url后加上exit为关闭

- 在url后加上restart为重启

- 在url后加上reload为重新加载配置信息



- [Jenkins（02）：Jenkins服务的关闭和重启](https://blog.csdn.net/qq_36396763/article/details/89501772?utm_medium=distribute.pc_relevant.none-task-blog-title-2&spm=1001.2101.3001.4242)