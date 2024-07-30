# 需求分析

目标：动态代码评审

1. 代码变化，git diff
2. open ai 【描述】对代码变化记录进行评审 【chatglm】
3. 通知 公众号测试平台

如何使用？

1. 引入sdk包， 调用方法（代码侵入性强，所有业务系统，都需要配置）
2. github actions CI&CD（免费云服务器） 配置系统 执行脚本 ， 执行main函数，拉取git diff， openai评审，消息触达

细节：

git diff：git diff -1 ～1

代码评审：脚本，模型 4o

评审记录：github 仓库log

# github acitons

