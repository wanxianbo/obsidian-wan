# Jenkins Centos 7 部署文档

## 备注说明：

1. 安装jenkins时要安装最新的版本，因为版本过低的话，会导致很多插件安装不上
2. 前提条件已经成功安装JDK

## 安装

1、cd进入到想要存放Jenkins文件的路径，接着创建一个用于存放Jenkins文件的文件夹，此处我们就创建一个名为jenkins的文件夹。输入命令：

2、cd 进入到新创建的jenkins目录下

3、添加Jenkins库到yum库，输入如下命令：

``` shell
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
# 证书问题：wget --no-check-certificate -0
```

4、输入如下命令，开始安装jenkins

​	下载安装指定版本，此处我们指定的版本为最新的jenkins-2.204.2-1.1.noarch.rpm，安装成功后，再解压

​	安装方法：

```shell
wget http://pkg.jenkins-ci.org/redhat-stable/jenkins-2.204.2-1.1.noarch.rpm
rpm -ivh jenkins-2.204.2-1.1.noarch.rpm
```

## 配置Jenkins

1、输入如下命令，进入端口配置文件

```shell
vim /etc/sysconfig/jenkins
# 找到 JENKINS_PORT="8080"
```

2、配置Jenkins的 JDK 环境变量

```shell
vim /etc/init.d/jenkins
# 可以建立软连接 ln -s /usr/local/java/jdk1.8.0_121/bin/java /usr/bin/java
```

## 启动Jenkins

1、输入如下命令，启动jenkins

```shell
sudo systemctl start jenkins
sudo service jenkins start 启动 
sudo service jenkins stop 停止
sudo service jenkins restart 重启
```

2、在shell窗口中输入如下命令，获得初始密码

```shell
cat /var/lib/jenkins/secrets/initialAdminPassword 
```

3、安装插件--点击“安装推荐的插件”

![image-20220503103242078](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/05/image-20220503103242078.png)

4、开始安装插件

​	**备注说明：**

（1）安装插件过程非常缓慢，估计要等待两个小时左右，如果有安装失败的插件，可以后面重新安装。

（2）建议一开始安装jenkins时要安装最新的版本，因为版本过低的话，会导致很多插件安装不上，亲测过。

5、如果中途出现安装失败的插件，可以点击“重试”，再继续安装

## 登录

1、创建管理员账号，输入账号和密码，此处我们就用admin/123456
![](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/05/image-20220503103444489.png)

2、登录成功后，可以看到画面如下所示，即表示安装成功

![image-20220503104312671](https://raw.githubusercontent.com/wanxianbo/pic-bed/main/img/2022/05/image-20220503104312671.png)