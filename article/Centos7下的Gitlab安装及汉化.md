# Centos7下的Gitlab安装及汉化

## 一、安装gitlab

#### 1.安装gitlab依赖

```
yum install -y curl policycoreutils-python openssh-server
systemctl enable sshd
systemctl start sshd
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
systemctl reload firewalld
```


#### 2.下载gitlab，rpm包

自选版本（清华镜像）：<https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/>

```
#下载12.0.3版本(汉化包也是12.0.3，两个要一样)
wget https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/gitlab-ce-12.3.9-ce.0.el7.x86_64.rpm
#安装gitlab
rpm -ivh gitlab-ce-12.3.9-ce.0.el7.x86_64.rpm
```


#### 3.修改配置文件

```
vim /etc/gitlab/gitlab.rb
external_url修改为你的ip或域名
```


#### 4.配置邮箱

```
vim /etc/gitlab/gitlab.rb
#52行左右
gitlab_rails['gitlab_email_enabled'] = true
gitlab_rails['gitlab_email_from'] = '742899387@qq.com'
gitlab_rails['gitlab_email_display_name'] = 'soulchild-gitlab'
#517行左右
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.qq.com"
gitlab_rails['smtp_port'] = 465
gitlab_rails['smtp_user_name'] = "你的邮箱"
gitlab_rails['smtp_password'] = "授权码"
gitlab_rails['smtp_domain'] = "qq.com"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = true
```


#### 5.重新生成配置文件并启动服务

```
gitlab-ctl reconfigure
gitlab-ctl status
```


#### 6.打开地址就可以访问了

## 二、汉化

没git先安装，yum -y install git

#### 1、下载汉化补丁

```
[root@gitlab ~]# git clone https://gitlab.com/xhang/gitlab.git
[root@gitlab ~]# cd gitlab
```



#### 2、查看全部分支版本

```
[root@gitlab ~]# git branch -a
```

#### 3、对比版本、生成补丁包

版本号前两位就好12-0，12-0-stable表示大版本内的稳定版，git diff 表示对比差异

```
[root@gitlab ~]# git diff remotes/origin/12-3-stable remotes/origin/12-3-stable-zh > /tmp/12.3.9-zh.diff
```

#### 4、停止服务器

```
[root@gitlab ~]# gitlab-ctl stop
```

#### 5、打补丁

patch指令让用户利用设置修补文件的方式，修改，更新原始文件。

```
#安装yum -y install patch
[root@gitlab ~]# patch -d /opt/gitlab/embedded/service/gitlab-rails -p1 < /tmp/12.3.9-zh.diff
```

PS：如果出现类似以下内容，则按住回车，一直跳过就行了

```
can't find file to patch at input line 5
Perhaps you used the wrong -p or --strip option?
The text leading up to this was:
--------------------------
|diff --git a/app/assets/javascripts/awards_handler.js b/app/assets/javascripts/awards_handler.js
|index eb0f06e..73e4833 100644
|--- a/app/assets/javascripts/awards_handler.js
|+++ b/app/assets/javascripts/awards_handler.js
--------------------------
File to patch:
```

#### 6、启动和重新配置

```
[root@gitlab ~]# gitlab-ctl start
[root@gitlab ~]# gitlab-ctl reconfigure
```



#### 7、如果汉化不彻底，在个人设置中，偏好设置， 选择 “简体中文” 即可