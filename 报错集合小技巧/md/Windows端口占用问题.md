# Windows解决端口被占用问题

解决办法：找到占用端口号的进程，关掉该进程，以下步骤以关闭占用9411端口的进程为例：

一、win+r打开cmd。

二、找到占用端口号的进程

```shell
$ netstat -ano|findstr "9411"

  TCP    127.0.0.1:53082        127.0.0.1:9411         SYN_SENT        13032
  TCP    127.0.0.1:53083        127.0.0.1:9411         SYN_SENT        13032
  TCP    127.0.0.1:53085        127.0.0.1:9411         SYN_SENT        13032
  TCP    [::1]:53077            [::1]:9411             SYN_SENT        13032
  TCP    [::1]:53078            [::1]:9411             SYN_SENT        13032
  TCP    [::1]:53081            [::1]:9411             SYN_SENT        13032
  
$ tasklist|findstr "13032"  

chrome.exe                   13032 Console                    1     51,888 K

$ tskill 13032
```

