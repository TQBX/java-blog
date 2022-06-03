近期，需要用python实现一个录音并保存为wav的功能，找到了这篇文章：[使用python怎么录音麦克风并生成wav文件](https://www.yisu.com/zixun/170694.html)

> python版本3.10

遇到的第一个问题：
![image-20220518125111017](img/mac%E5%AE%89%E8%A3%85pyaudio%E8%B8%A9%E5%9D%91/image-20220518125111017.png)

执行如下指令即可：

```bash
python3 -m pip install --upgrade pip -i https://pypi.douban.com/simple
```

接下来执行

```bash
pip install pyaudio -i http://pypi.douban.com/simple --trusted-host pypi.douban.com
```

遇到的第二个问题：

![image-20220518125357243](img/mac%E5%AE%89%E8%A3%85pyaudio%E8%B8%A9%E5%9D%91/image-20220518125357243.png)

找了许多的解决方案，mac上可以利用`brew install portaudio `去安装portaudio，也可以去portaudio官网去下载编译，于是我选择了安装brew，找到了这篇文章：https://www.jianshu.com/p/06a9a59e7040

安装完成之后，再次执行上面pip install的指令，还是不行，很奇怪啊，明明portaudio已经下载了啊。我是下载在`/opt/homebrew/Cellar/portaudio/19.7.0`路径下的。

然后找到有说法是没有安装xcode，于是我又执行了`xcode-select --install`（提示我已经安装了）

兜兜转转，最终找的了这篇文章：https://blog.csdn.net/XB_please/article/details/121495325

解决方案是：

```bash
python3 -m pip install --global-option='build_ext' --global-option='-I/opt/homebrew/Cellar/portaudio/19.7.0/include' --global-option='-L/opt/homebrew/Cellar/portaudio/19.7.0/lib' pyaudio
```

这里include 和 lib的路径记得自己修改，接着在install就大功告成了。

![image-20220518125917595](img/mac%E5%AE%89%E8%A3%85pyaudio%E8%B8%A9%E5%9D%91/image-20220518125917595.png)