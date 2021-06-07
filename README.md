# docker环境部署

## linux安装

建议CentOS7，安装方法略。

## docker环境安装

### docker安装
1. 安装yum-utils：

  ```shell
  yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
  ```

2. 为yum源添加docker仓库位置：

  ```shell
  yum-config-manager \
  --add-repo \
  https://download.docker.com/linux/centos/docker-ce.repo
  ```

3. 安装docker:

  ```shell
  yum install docker-ce
  ```

4. 启动docker:


  ```shell
  systemctl start docker
  systemctl enable docker
  ```

5. 安装上传下载插件：

  ```shell
  yum -y install lrzsz
  ```
### Docker Registry 2.0搭建

```shell
docker run -d -p 5000:5000 --restart=always --name registry2 registry:2
```

如果遇到镜像下载不下来的情况，需要修改 /etc/docker/daemon.json 文件并添加上 registry-mirrors 键值，然后重启docker服务：

```json
{
    "registry-mirrors": ["https://registry.docker-cn.com"]
}
```

### Docker开启远程API

#### 用vim编辑器修改docker.service文件

```shell
vim /usr/lib/systemd/system/docker.service
```

需要修改的部分：

```shell
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```

修改后的部分：

```shell
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock
```

#### 让Docker支持http上传镜像

```shell
echo '{ "insecure-registries":["10.2.33.155:5000"] }' > /etc/docker/daemon.json
```

*注：ip为自己服务器的。也可直接修改daemon.json文件`vim /etc/docker/daemon.json`如下：*

```json
{
    "registry-mirrors": [
        "https://registry.docker-cn.com",
        "https://hub-mirror.c.163.com",
        "https://dockerhub.azk8s.cn", 
        "http://10.2.33.155:5000"
    ],
 
    "insecure-registries": [
        "localhost:5000",
        "10.2.33.155:5000"
    ],
    
    "data-root": "/data/docker" # 指定docker数据目录，用于挂载数据盘时
}
```

#### 修改配置后需要使用如下命令使配置生效

```shell
systemctl daemon-reload
```

#### 重新启动Docker服务

```shell
systemctl restart docker
```

#### 开启防火墙的Docker构建端口

```shell
firewall-cmd --zone=public --add-port=2375/tcp --permanent
firewall-cmd --reload
```

### docker compose安装

1. 下载地址：https://github.com/docker/compose/releases 
2. 安装地址：/usr/local/bin/docker-compose

   命令如下：

```shell
curl -L https://get.daocloud.io/docker/compose/releases/download/1.28.2/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
```

2. 设置为可执行：

```shell
sudo chmod +x /usr/local/bin/docker-compose
```

3. 测试是否安装成功：

```shell
docker-compose --version
```

### 可视化管理工具

> Portainer 是一款轻量级的应用，它提供了图形化界面，用于方便的管理Docker环境，包括单机环境和集群环境，下面我们将用Portainer来管理Docker容器中的应用。

- 官网地址：https://github.com/portainer/portainer

- 获取Docker镜像文件：

```shell
docker pull portainer/portainer
```

- 使用docker容器运行Portainer：

```shell
docker run -p 9000:9000 -p 8000:8000 --name portainer \
--restart=always \
-v /var/run/docker.sock:/var/run/docker.sock \
-v /data/portainer/data:/data \
-d portainer/portainer
```

- 放开端口：

```shell
firewall-cmd --zone=public --add-port=9000/tcp --permanent
firewall-cmd --reload
```

- 查看Portainer的DashBoard信息：http://10.2.33.155:9000

## 下载部署magento2

### 在线执行

```shell
curl -s https://raw.githubusercontent.com/Rean1030/docker4magento/master/onelinesetup | bash -s -- magento.test 2.4.2 community developer
```

### 本地执行

将代码完全下载到要启动的环境，进入项目根目录执行下面命令：

```shell
./localsetup magento.test 2.4.2 community developer
```

其中
* 第一个参数（magento.test）是本地host地址
* 第二个参数（2.4.2）是版本
* 第三个参数（community）表示下载的代码版本[community, enterprise]
* 第四个参数（developer）为启动模式[maintenance, developer, production, default]，其中开发模式会开启xdebug调试，各模式差异详情参考官方文档

数据库配置在`compose/env/db.env`中配置，magento配置在`compose/env/magento.env`中配置

### 压缩包下载

上面的方法可能会出现访问不了，我们需要先直接整包下载后解压：

```shell
curl -o docker4magento.zip  https://github.com/Rean1030/docker4magento/archive/master.zip
unzip docker4magento.zip
cd docker4magento
```
### 其他操作

* 如果提示權限不足，修改文件执行权限

```shell
chmod -R +x compose/bin/
chmod +x buildimages localsetup onelinesetup template
```

* 转换文件结尾（如果执行遇到/bin/sh ^M: bad interpreter: No such file or directory）

```shell
find -type f | xargs dos2unix -o
```

* 下载国外镜像的源码比较慢可以直接走官网下载方式，需要装composer

先安装php
```shell
yum install php
yum install php-json
```

再装composer
```shell
curl -sS https://getcomposer.org/installer | php
mv composer.phar  /usr/local/bin/composer
或
wget https://dl.laravel-china.org/composer.phar -O /usr/local/bin/composer
chmod a+x /usr/local/bin/composer
```

composer安装也会比较慢，如果切国内镜像又获取不到包，解决办法是直接[下载源码压缩包](https://github.com/magento/magento2/releases/)，放到脚本解压后的docker4magento目录，再执行安装命令，此时下载程序判断已经下载了就可以直接用下载的包解压打包了

* es设置
需要设置系统内核参数，否则会因为内存不足无法启动。
```shell
# 改变设置
sysctl -w vm.max_map_count=262144
# 使之立即生效
sysctl -p
```
* ipv4转发开启(如果报 WARNING：IPv4 forwarding is disabled. Networking will not work.)
```shell
vi /etc/sysctl.conf
net.ipv4.ip_forward=1  #修改此处，没有则添加

systemctl restart network && systemctl restart docker # 报没有network服务，可以用[ifdown eth0 && ifup eth0 && systemctl restart docker]
sysctl net.ipv4.ip_forward  # 验证结果
```

* 设置系统参数（Error: ENOSPC: System limit for number of file watchers reached, watch）
```shell
echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf && sudo sysctl -p
sudo sysctl --system
```

* 如果启动登陆管理界面提示必须2fa设置，可以先关闭，进去设置完后打开

```shell
bin/magento module:disable Magento_TwoFactorAuth
bin/magento module:enable Magento_TwoFactorAuth
```

或者为账号设置两步验证，以谷歌为例，手机安装谷歌验证app，magento中如下设置：

```shell
Select Google Authenticator as the 2FA provider:
bin/magento config:set twofactorauth/general/force_providers google

Increase the lifetime of the window to 60 seconds to prevent tokens from expiring.
bin/magento config:set twofactorauth/google/otp_window 60

Generate a Base32-encoded string for the shared secret value. For example, encoding the string abcd with the online [Base32](https://emn178.github.io/online-tools/base32_encode.html) Encode tool returns the value MFRGGZDF. Use the following key to add the encoded value to the MFTF .credentials file(cd vendor/magento/magento2-functional-testing-framework/etc/config/, cp .credentials.example .credentials):
magento/tfa/OTP_SHARED_SECRET=MFRGGZDF

Add the encoded shared secret to Google Authenticator.
bin/magento security:tfa:google:set-secret admin MFRGGZDF

```

再继续进行上面的[本地执行](https://github.com/Rean1030/docker4magento#%E6%9C%AC%E5%9C%B0%E6%89%A7%E8%A1%8C)

## 常用命令

```shell
bin/magento module:enable --all
bin/magento setup:upgrade
bin/magento setup:di:compile
bin/magento cache:clean
bin/magento cache:flush
```