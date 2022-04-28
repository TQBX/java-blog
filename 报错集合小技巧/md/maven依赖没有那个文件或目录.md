【已解决】maven导入依赖 出现问题：spring-boot-starter-parent-2.6.7.pom.part.lock (No such file or directory)

解决办法：mac 修改权限：chmod -R 777 /usr/local/maven-repo

【已解决】Plugin ‘org.springframework.boot:spring-boot-maven-plugin:‘ not found

解决办法：加入`<version>${project.parent.version}</version>`

![image-20220428121853402](img/maven%E4%BE%9D%E8%B5%96%E6%B2%A1%E6%9C%89%E9%82%A3%E4%B8%AA%E6%96%87%E4%BB%B6%E6%88%96%E7%9B%AE%E5%BD%95/image-20220428121853402.png)