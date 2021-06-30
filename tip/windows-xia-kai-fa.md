# Windows+虚拟机+Docker

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

```text
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

坑：使用多行\(\)时，注意是否有空格，否则命令会失效

### 安装Docker

[https://docs.docker.com/engine/install/ubuntu/](https://docs.docker.com/engine/install/ubuntu/)

```text
# 老版本
sudo apt-get install docker.io
sudo apt-get remove docker docker-engine docker.io containerd runc

# 新版本
sudo apt-get update;
sudo apt-get upgrade;
sudo apt-get install net-tools ssh apt-transport-https ca-certificates curl gnupg lsb-release;

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update;
sudo apt-get install docker-ce docker-ce-cli containerd.io

# 验证安装成功
$ docker version
# 或者
$ docker info
```

### Docker可视化管理工具Portainer

[https://documentation.portainer.io/v2.0/deploy/ceinstalldocker/](https://documentation.portainer.io/v2.0/deploy/ceinstalldocker/)

```bash
docker volume create portainer_data
docker run -d -p 8000:8000 -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce
```

### 安装常用软件

注：可以通过`docker stats`查看默认运行的大小，然后通过-m加上限制

### MySQL\(3306:3306\)

root/admin

```bash
docker run --name test-mysql -d \
-m 512M --memory-swap 1G \
-p 3306:3306 \
--restart always \
-v /usr/loca/data/mysql/conf:/etc/mysql/conf.d -e MYSQL_ROOT_PASSWORD=admin mysql:5.7
```

### MongoDB\(27017:27017\)

admin/admin

```bash
docker run --name test-mongo -d \
-m 256M --memory-swap 512M \
-p 27017:27017 \
--restart always \
-e MONGO_INITDB_ROOT_USERNAME=admin -e MONGO_INITDB_ROOT_PASSWORD=admin mongo
```

### Redis\(6379:6379\)

```bash
docker run --name test-redis -d \
-m 256M --memory-swap 512M \
-p 6379:6379 \
--restart always \
redis redis-server --appendonly yes

# 虚拟机安装redis客户端
$ sudo apt install redis-tools
```

### Nginx\(9001:80\)\(8889:8889\)

admin/admin

```bash
sudo mkdir -p /usr/local/data/nginx/conf/;
sudo mkdir -p /usr/local/data/nginx/html/;
sudo mkdir -p /usr/local/data/nginx/logs/;
sudo touch /usr/local/data/nginx/conf/default.conf;


docker run --name nginx -d \
-p 80:80 \
-v /usr/local/data/nginx/conf/default.conf:/etc/nginx/conf.d/default.conf \
-v /usr/local/data/nginx/conf/nginx.conf:/etc/nginx/nginx.conf \
-v /usr/local/data/nginx/html:/usr/share/nginx/html \
-v /usr/local/data/nginx/logs:/var/log/nginx \
nginx:1.19.7

sever {
    location /{
        root /usr/local/data/nginx/html;
        autoindex on; # 开启目录浏览功能
        autoindex_localtime on;
    }
}

# nginx-gui
docker pull crazyleojay/nginx_ui

docker run --detach \
--publish 9001:80 --publish 8889:8889 \
--name nginx_ui \
--restart always \
crazyleojay/nginx_ui:latest
```

### GitLab\(9002:80\)

root/12345678

[GitLab Docker images \| GitLab](https://docs.gitlab.com/omnibus/docker/)

[https://hub.docker.com/r/gitlab/gitlab-ee](https://hub.docker.com/r/gitlab/gitlab-ee)

```bash
# git-ce 是社区版，gitlab-ee是企业版，收费版
$ docker pull gitlab/gitlab-ce
export GITLAB_HOME=/usr/local/data/gitlab

# 注意，gitlab特别吃内存，这里仅是本地搭建，设置成了--restart no
docker run --detach \
  --hostname gitlab.example.com \
  --publish 443:443 --publish 9002:80 --publish 222:22 \
  --name gitlab \
  --restart no \
  --volume $GITLAB_HOME/config:/etc/gitlab \
  --volume $GITLAB_HOME/logs:/var/log/gitlab \
  --volume $GITLAB_HOME/data:/var/opt/gitlab \
  gitlab/gitlab-ce
```

使用：新建组-新建工程（在组下）-新建用户（分配组，赋值密码）

### spug\(9003:80\)

admin/admin

[https://spug.dev/docs/install-docker/](https://spug.dev/docs/install-docker/)

```bash
docker pull registry.aliyuncs.com/openspug/spug
docker run -d --name=spug \
-p 9003:80 \
--restart always \
-v /usr/local/data/spug:/data registry.aliyuncs.com/openspug/spug

docker exec spug init_spug admin admin
docker restart spug
```

### xxl-job\(9004:8080\)

访问：[http://localhost:8080/xxl-job-admin](http://localhost:8080/xxl-job-admin)

[https://www.xuxueli.com/xxl-job/\#%E4%BA%8C%E3%80%81%E5%BF%AB%E9%80%9F%E5%85%A5%E9%97%A8](https://www.xuxueli.com/xxl-job/#%E4%BA%8C%E3%80%81%E5%BF%AB%E9%80%9F%E5%85%A5%E9%97%A8)

```bash
# 下载数据库脚本执行
https://gitee.com/xuxueli0323/xxl-job/blob/master/doc/db/tables_xxl_job.sql#
# 注：需要下载配置文件，修改数据库连接信息
wget https://raw.githubusercontent.com/xuxueli/xxl-job/2.3.0/xxl-job-admin/src/main/resources/application.properties

docker run -d --name xxl-job-admin \
-p 9004:8080 \
--restart always \
-v /usr/local/data/application.properties:/application.properties \
-e PARAMS='--spring.config.location=/application.properties' xuxueli/xxl-job-admin:2.3.0

# 本地访问
--net host \
```

### Cloudreve云盘系统\(5212:5212\)

首次启动后请执行docker logs -f cloudreve获取初始密码 [https://hub.docker.com/r/xavierniu/cloudreve](https://hub.docker.com/r/xavierniu/cloudreve)

```bash
sudo mkdir -p /home/ubuntu/data/cloudreve/uploads;
sudo mkdir -p /home/ubuntu/data/conf.ini;
sudo mkdir -p /home/ubuntu/data/cloudreve.db;
sudo mkdir -p /home/ubuntu/data/cloudreve/avatar;

sudo docker run -d --name cloudreve \
-e PUID 1000 \
-e PGID 1000 \
-e TZ "Asia/Shanghai" \
-p 5212:5212 \
--restart always \
-v /home/ubuntu/data/cloudreve/uploads:/cloudreve/uploads \
-v /home/ubuntu/data/cloudreve/conf.ini:/cloudreve/conf.ini \
-v /home/ubuntu/data/cloudreve/cloudreve.db:/cloudreve/cloudreve.db \
-v /home/ubuntu/data/cloudreve/avatar:/cloudreve/avatar xavierniu/cloudreve
# 首次启动后请执行docker logs -f cloudreve获取初始密码
```

### FileRun云盘系统\(9005:80\)

superuser/superuser [https://docs.filerun.com/docker](https://docs.filerun.com/docker)

```bash
sudo mkdir -p /home/ubuntu/data/filerun/db;
sudo mkdir -p /home/ubuntu/data/filerun/html;
sudo mkdir -p /home/ubuntu/data/filerun/user-files;
sudo touch docker-compose.yml;
sudo vim docker-compose.yml;
```

创建一个名称为docker-compose.yml的文件

```text
version: '2'

services:
  db:
    image: mariadb:10.1
    environment:
      MYSQL_ROOT_PASSWORD: admin # your_mysql_root_password
      MYSQL_USER: root # your_filerun_username
      MYSQL_PASSWORD: admin # your_filerun_password
      MYSQL_DATABASE: file_run # your_filerun_database
    volumes:
      - /home/ubuntu/data/filerun/db:/var/lib/mysql

  web:
    image: afian/filerun
    environment:
      FR_DB_HOST: db
      FR_DB_PORT: 3306
      FR_DB_NAME: file_run # your_filerun_database
      FR_DB_USER: root # your_filerun_username
      FR_DB_PASS: admin # your_filerun_password
      APACHE_RUN_USER: www-data
      APACHE_RUN_USER_ID: 33
      APACHE_RUN_GROUP: www-data
      APACHE_RUN_GROUP_ID: 33
    depends_on:
      - db
    links:
      - db:db
    ports:
      - "9005:80"
    volumes:
      - /home/ubuntu/data/filerun/html:/var/www/html
      - /home/ubuntu/data/filerun/user-files:/user-files
```

启动

```text
docker-compose up -d
```

