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

