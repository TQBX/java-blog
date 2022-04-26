1. 打开终端
2. 停止mysql服务器`sudo /usr/local/mysql/support-files/mysql.server stop`

3. `cd /usr/local/mysql/bin`

4. `sudo su`获取权限，提示输入mac登陆密码
5. `./mysqld_safe --skip-grant-tables &`跳过安全认证
6. 测试`./mysql`，出现`Welcome to the MySQL monitor.`等字样
7. 设置密码，8.04之后的版本修改密码如下：
   1. `use mysql;`
   2. `ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '新密码';`
   3. `FLUSH PRIVILEGES;`

8. 重启mysql即可，`mysql -u root -p 新密码`

