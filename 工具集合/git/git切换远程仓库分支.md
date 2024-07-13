### 切换远程仓库地址

一、URL为新的url地址

```shell
git remote set-url origin URL 
```

二、先删除远程仓库地址，再设置

```shell
git remote rm origin
git remote add origin URL
```

### 查看远程仓库地址

```shell
git remote -v
```

## GitHub token远程连接

### 生成token

settings->developer settings ->personal access tokens-generate new token

### 重新设置远程仓库地址

```sh
git remote rm origin
git remote add origin https://TQBX:【token】@github.com/TQBX/java-blog.git
```

## main提交

```sh
git branch -M main
git push -u origin main
```

