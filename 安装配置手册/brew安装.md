安装教程：https://www.jianshu.com/p/06a9a59e7040

```
which ruby
ruby --version
```

```bash
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
or
/bin/zsh -c "$(curl -fsSL https://gitee.com/cunkai/HomebrewCN/raw/master/Homebrew.sh)"
```

```
本地软件库列表：brew ls
查找软件：brew search google（其中google替换为要查找的关键字）
查看brew版本：brew -v  更新brew版本：brew update
安装cask软件：brew install --cask firefox 把firefox换成你要安装
```

```
brew --version
which brew
```



brew遇到的问题：使用homebrew 的时候失败fatal: not in a git directory Error: Command failed with exit 128: git

![image-20220610114638036](img/brew%E5%AE%89%E8%A3%85/image-20220610114638036.png)

解决办法：

执行`brew -v` 查看提示，homebrew-core和homebrew-cask目录 被git认为不是一个安全的目录，需要两行命令添加。

```bash
huayuhao@tqbx-mbp ~ % brew -v
Homebrew 3.4.11-53-g34dd8e3
fatal: unsafe repository ('/opt/homebrew/Library/Taps/homebrew/homebrew-core' is owned by someone else)
To add an exception for this directory, call:

	git config --global --add safe.directory /opt/homebrew/Library/Taps/homebrew/homebrew-core
Homebrew/homebrew-core (no Git repository)
fatal: unsafe repository ('/opt/homebrew/Library/Taps/homebrew/homebrew-cask' is owned by someone else)
To add an exception for this directory, call:

	git config --global --add safe.directory /opt/homebrew/Library/Taps/homebrew/homebrew-cask
Homebrew/homebrew-cask (no Git repository)
```

根据提示，执行命令：

```bash
git config --global --add safe.directory /opt/homebrew/Library/Taps/homebrew/homebrew-cor
git config --global --add safe.directory /opt/homebrew/Library/Taps/homebrew/homebrew-cask
```

![image-20220610114838127](img/brew%E5%AE%89%E8%A3%85/image-20220610114838127.png)

至此，成功解决问题。
