# openstack贡献补充

这里主要补充官方文档之外的信息。请先阅读官方文档，然后阅读本文档，了解补充点后开始实验。


# 环境准备及练习
请先阅读官方文档
- https://docs.openstack.org/contributor-guide/quickstart/first-timers.html

实验环境，openstack提供实验项目给新手使用
- https://docs.openstack.org/infra/manual/sandbox.html


# 补充设置review
中华大地的墙很厉害，而 review 的默认协议为 ssh，为了绕开墙，我们需要换一种墙能认识的协议 https。 
## gerrit生成http password
生成 http 密码
在gerrit的个人设置界面，有 "http password"的选项，进入，点击生成，以下为文档参考
- https://review.openstack.org/Documentation/user-upload.html#http

## 设置http review 
以 sandbox 为例子

> git clone https://git.openstack.org/openstack-dev/sandbox

> cd sandbox

如果没有设置全局用户信息，则需要设置用户信息

> git config user.name {username}

> git config user.email {useremail}

设置review的用户信息

> git config gitreview.username {username}

修改gerrit协议到https

> git config gitreview.scheme https

> git config gitreview.port 443

添加gerrit仓库，使用gerrit生成的http password

> git remote add gerrit https://{username}:{htppasswd}@review.openstack.org:443/openstack-dev/sandbox.git

配置review环境

> git review -s -v

进行修改并提交
> git co -b fix-test

> cat > fix-test << EOF\nthis is first test for commit for openstack.\nEOF

> git st

> git add fix-test

> git commit

发起review
> git review





