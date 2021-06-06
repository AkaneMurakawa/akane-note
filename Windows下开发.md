# Windows下搭建Linux虚拟容器开发环境

## 安装虚拟机

略

## 使用X shell连接虚拟机

第一步：首先确保有网络，如果有图形界面可上网试一下。如果没有图形界面，可以用ping命令，例如：`ping bing.com`。在这里网络我选择的是**Bridged模式**。

第二步：使用`sudo apt-get update`更新依赖包，接着按照提示，安装`net-tools`，命令：`sudo apt install net-tools`

第三步：`ifconfig`

### **Connection failed.**

第四步：相互ping一下，可以发现虚拟机无法ping通本机

第五步：虚拟机安装ssh，命令：`sudo apt install ssh`

### **root无法连接**

```
sudo apt -y install vim
sudo vim /etc/ssh/sshd_config

搜索:/PermitRootLogin 
# 按照i，增加一行，填写 PermitRootLogin yes
按ESC 输入: wq!

重启ssh
service ssh restart

修改root密码
sudo passwd root
```



## Docker

### 安装Docker

`sudo apt install docker.io`

### Docker可视化管理工具Portainer

https://documentation.portainer.io/v2.0/deploy/ceinstalldocker/

```bash
docker volume create portainer_data
docker run -d -p 8000:8000 -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce
```



### 安装常用软件

```bash
#首次启动
$ docker run --name mysql -d -p 3306:3306 -e MYSQL_ROOT_PASSWORD=admin mysql:5.7
$ docker run --name redis -d -p 6379:6379 redis redis-server --appendonly yes
$ docker run --name mongo -d -p 27017:27017 -e MONGO_INITDB_ROOT_USERNAME=admin -e MONGO_INITDB_ROOT_PASSWORD=admin mongo
$ docker run --name nginx -d -p 8080:80 nginx:1.19.7


# 后续启动
$ docker start mysql
$ docker start redis
$ docker start mongo
$ docker start nginx

# 虚拟机安装redis客户端
$ sudo apt install redis-tools
```

### GitLab(默认端口80)

[GitLab Docker images | GitLab](https://docs.gitlab.com/omnibus/docker/)

https://hub.docker.com/r/gitlab/gitlab-ee

```
# git-ce 是社区版，gitlab-ee是企业版，收费版
$ docker pull gitlab/gitlab-ce
export GITLAB_HOME=/srv/gitlab

# 注意，gitlab特别吃内存
sudo docker run --detach \
  --hostname gitlab.example.com \
  --publish 443:443 --publish 9999:80 --publish 222:22 \
  --name gitlab \
  --restart always \
  --volume $GITLAB_HOME/config:/etc/gitlab \
  --volume $GITLAB_HOME/logs:/var/log/gitlab \
  --volume $GITLAB_HOME/data:/var/opt/gitlab \
  gitlab/gitlab-ce
```

新建组-新建工程（在组下）-新建用户（分配组，赋值密码）



### xxl-job

https://www.xuxueli.com/xxl-job/#%E4%BA%8C%E3%80%81%E5%BF%AB%E9%80%9F%E5%85%A5%E9%97%A8
```
https://gitee.com/xuxueli0323/xxl-job/blob/master/doc/db/tables_xxl_job.sql#

wget https://raw.githubusercontent.com/xuxueli/xxl-job/2.3.0/xxl-job-admin/src/main/resources/application.properties

docker run -d --name xxl-job-admin -p 8080:8080 -v /root/local/application.properties:/application.properties  --net host -e PARAMS='--spring.config.location=/application.properties' xuxueli/xxl-job-admin:2.3.0
```

### spug(默认端口80)
https://spug.dev/docs/install-docker/
```
docker pull registry.aliyuncs.com/openspug/spug
docker run -d --restart=always --name=spug -p 8888:80 -v /root/local/data/:/data registry.aliyuncs.com/openspug/spug
docker exec spug init_spug admin admin
docker restart spug
```

