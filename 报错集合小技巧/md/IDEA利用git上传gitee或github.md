# IDEA利用git上传项目到gitee或gitHub

之前一直使用github传传学习过程中的小demo，小文章，小笔记啥的，经常会上传失败。最近看到码云这个地方也很不错，主要是速度太快了，体验很好，特地记录一下这个过程，毕竟每次用没次到处找太麻烦了，干脆一起把遇到的问题都总结一下。

![新建仓库](../pic/新建仓库.png)



![创建项目](../pic/创建项目.png)



![image-20200315162518821](../pic/image-20200315162518821.png)

选择自己的想要上传的项目

![项目于](../pic/项目于.png)



![add5](../pic/add5.png)

![cimmit](../pic/cimmit.png)

![osuh](../pic/osuh.png)

![remote](../pic/remote.png)

![clone](../pic/clone.png)

即可。

# 出现问题push to origin/master was rejected

- 进入项目所在目录，git bash here

- 依次输入：

  ```git
  git pull
  git pull origin master
  git pull origin master --allow-unrelated-histories
  ```

- 重新push即可。

