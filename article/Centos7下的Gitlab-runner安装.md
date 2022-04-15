# Centos7下的Gitlab-runner安装

**一、安装runnr**

自选版本（清华镜像）：<https://mirrors.tuna.tsinghua.edu.cn/gitlab-runner/yum/el7/>

```
# 下载gitlab-runner
wget https://mirrors.tuna.tsinghua.edu.cn/gitlab-runner/yum/el7/gitlab-runner-13.1.1-1.x86_64.rpm
# 安装
rpm -ivh gitlab-runner-13.1.1-1.x86_64.rpm
# 检查安装是否成功
gitlab-runner --version
```

**二** **、** **配置runner**

默认情况，runner是通过gitlab-runner的这个用户来执行一系列操作，其工作目录也是在gitlab-runner的用户目录下面。如果使用默认gitlab-runner用户操作一些文件时经常会遇到权限问题，就需要给gitlab-runner赋权。

```
# 帮助
gitlab-runner -h

# 注册runner，
# 先打开GitLab上需要自动部署的项目界面，
# 找到该项目的Settings –> CI/CD –> Runners settings 在gitlab上可以看到自己的token信息。
gitlab-runner register 

# 注册步骤
Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/):
http://gitlab.xiaoying.love/
Please enter the gitlab-ci token for this runner:
HyxnLB-N8ReREP9_MPBz
Please enter the gitlab-ci description for this runner:
[VM_0_4_centos]: test-runner
Please enter the gitlab-ci tags for this runner (comma separated):
test-runner
Registering runner... succeeded                     runner=HyxnLB-N
Please enter the executor: custom, docker, shell, docker+machine, docker-ssh+machine, docker-ssh, parallels, ssh, virtualbox, kubernetes:
shell
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded! 


# 查看状态
gitlab-runner status

# 重启服务
gitlab-runner restart
```

**三** **、** **问题的解决**

**1.提示报错没有这些node，npm，pm2命令**

runner执行gitlab-ci.yml文件的script部分时报错没有上述这些命令

原因可能是ssh进入服务器的种类很多，无法获取的环境变量

还有种可能是runner执行的时候是gitlab-runner这个用户，无法获取最高权限root

解决：

-   使用env命令显示所有的环境变量，执行shell脚本前加上

```
#全局环境变量
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
```

-   把gitlab-runner用户添加到root用户组里面

**2.gitlab-runner如何更改执行用户**

通过指令`ps aux|grep gitlab-runner`可以看到：

```
/usr/bin/gitlab-ci-multi-runner run --working-directory /home/gitlab-runner --config /etc/gitlab-runner/config.toml --service gitlab-runner --syslog --user gitlab-runner
```

其中：

`--working-directory`：设置工作目录, 默认是 **/home/{执行user}**

`--config`：设置配置文件目录，默认是 **/etc/gitlab-runner/config.toml**

`--user`：设置执行用户名，默认是**gitlab-runner**

****

因此想要更改`user`为`root`只需要重新设置`--user`属性即可，步骤如下：

1.删除`gitlab-runner`

```
sudo gitlab-runner uninstall
```

2.安装并设置`--user`(例如我想设置为root)

```
gitlab-runner install --working-directory /home/gitlab-runner --user root
```

3.重启`gitlab-runner`

```
sudo service gitlab-runner restart
```

验证一下：

再次执行`ps aux|grep gitlab-runner`会发现`--user`的用户名已经更换成`root`了

```
/usr/bin/gitlab-ci-multi-runner run --working-directory /home/gitlab-runner --config /etc/gitlab-runner/config.toml --service gitlab-runner --syslog --user root
```

至此gitlab-runner执行`.gitlab-cli.yaml`时候便是以`root`用户去执行操作，再也没有繁琐的权限问题了

**3.** **SSH基础知识以及配置免密登录（gitlab-runner为例）**

如果想通gitlab-runner从A服务器ssh（登录）到B服务器。

看下面链接！！！！

**链接：** <https://blog.csdn.net/frankcheng5143/article/details/86062867>