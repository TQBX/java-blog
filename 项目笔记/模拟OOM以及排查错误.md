配置：Mac M1

# Mac M1 Visual VM下载

## 下载

https://visualvm.github.io/download.html

下载地址，安装

## 修改jdk path

找到本机的java home，复制

```shell
echo $JAVA_HOME
```

打开visualvm.conf文件

```shell
vim /Applications/VisualVM.app/Contents/Resources/visualvm/etc/visualvm.conf
```

修改最后一行

```shell
visualvm_jdkhome="刚刚复制的java_home"
```

## 启动打开

提示“Visual VM”已损坏，无法打开。您应该将它移到废纸篓

在终端输入命令：

```shell
sudo xattr -r -d com.apple.quarantine /Applications/VisualVM.app/
```

## 报错

OSError: [Errno 1] Operation not permitted， 查找资料，是因为M1开启了SIP，关闭即可

参考：https://www.bilibili.com/read/cv32771317/

之后 继续

```shell
sudo xattr -r -d com.apple.quarantine /Applications/VisualVM.app/
```

![image-20240724200949795](img/%E6%A8%A1%E6%8B%9FOOM%E4%BB%A5%E5%8F%8A%E6%8E%92%E6%9F%A5%E9%94%99%E8%AF%AF/image-20240724200949795.png)

# Mac M1 MAT下载

下载地址：[https://www.eclipse.org/mat/downloads.php](https://eclipse.dev/mat/downloads.php?spm=a2c6h.12873639.article-detail.7.6f4c300cGmkV9T)

![image-20240724192421430](img/%E6%A8%A1%E6%8B%9FOOM%E4%BB%A5%E5%8F%8A%E6%8E%92%E6%9F%A5%E9%94%99%E8%AF%AF/image-20240724192421430.png)

打开，如果提示不安全，可以系统设置，隐私与安全性，确认打开

![image-20240724201603567](img/%E6%A8%A1%E6%8B%9FOOM%E4%BB%A5%E5%8F%8A%E6%8E%92%E6%9F%A5%E9%94%99%E8%AF%AF/image-20240724201603567.png)

# Mac M1 VisualGC下载

打开Tools，Plugins，下载Visual GC插件

![image-20240724194919823](img/%E6%A8%A1%E6%8B%9FOOM%E4%BB%A5%E5%8F%8A%E6%8E%92%E6%9F%A5%E9%94%99%E8%AF%AF/image-20240724194919823.png)

启动OOMTest.java

![image-20240724200825791](img/%E6%A8%A1%E6%8B%9FOOM%E4%BB%A5%E5%8F%8A%E6%8E%92%E6%9F%A5%E9%94%99%E8%AF%AF/image-20240724200825791.png)

元空间 Metaspace、编译时间、类加载时间，

GC time：垃圾回收了85次，最后一次回收的原因是分配失败

eden：已使用22m，共65m。survivor0和1， old gen老年代

# 模拟Java OOM

## 编写代码

```java
import java.util.ArrayList;

/**
 * <pre>{@code
 *
 * }</pre>
 *
 * @author Summerday
 * @date 2024/7/24
 */
public class OOMTest {
    static class Client {
        Long idx;

        Client(Long idx) {
            this.idx = idx;
        }
    }

    public static void main(String[] args) {
        ArrayList<Client> clients = new ArrayList<>();
        long idx = 0;
        while(true) {
            clients.add(new Client(idx));
            idx++;
            Runtime run = Runtime.getRuntime();
            long max = run.maxMemory() / 1024 / 1024;
            long total = run.totalMemory() / 1024 / 1024;
            long free = run.freeMemory() / 1024 / 1024;
            long usable = max - total + free;
            System.out.println("最大内存 = " + max + "M");
            System.out.println("已经分配内存 = " + total + "M");
            System.out.println("已经分配内存中的最大空间 = " + free + "M");
            System.out.println("最大可用内存 = " + usable + "M");
            int size = clients.size();
            System.out.println("client size = " + size);
            if(idx == 20) {
                break;
            }
        }
    }
}
```

## 修改IDEA虚拟机参数

![image-20240724191420400](img/%E6%A8%A1%E6%8B%9FOOM%E4%BB%A5%E5%8F%8A%E6%8E%92%E6%9F%A5%E9%94%99%E8%AF%AF/image-20240724191420400.png)

![image-20240724191459314](img/%E6%A8%A1%E6%8B%9FOOM%E4%BB%A5%E5%8F%8A%E6%8E%92%E6%9F%A5%E9%94%99%E8%AF%AF/image-20240724191459314.png)

添加参数

```shell
-Xms200m -Xmx200m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=./my_java_heapdump.hprof
```

初始堆内存大小200M，最大堆内存大小200M，

## 启动程序

如期的结果：

![image-20240724192109837](img/%E6%A8%A1%E6%8B%9FOOM%E4%BB%A5%E5%8F%8A%E6%8E%92%E6%9F%A5%E9%94%99%E8%AF%AF/image-20240724192109837.png)

并且在项目路径下，生成了./my_java_heapdump.hprof文件

> dump文件获取方式：主动or被动
>
> 被动：设置-XX:+HeapDumpOnOutOfMemoryError， OOM事件
>
> 主动：
>
> - map -dump:[live],format=b,file=
> - jcmd GC.heap_dump
> - VisualVM 的monitor 生成dump文件
> - jmx
>
> 