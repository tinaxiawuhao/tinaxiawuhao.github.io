---
title: 'gitlab使用runner完成CI'
date: 2022-09-08 19:29:52
tags: [git]
published: true
hideInList: false
feature: /post-images/0SH2PgZRJ.png
isTop: false
---


### gitlab安装

#### docker下载镜像

```shell
docker pull gitlab/gitlab-ce:latest
```

#### gitlab容器部署

```shell
docker run -d -p 8443:443 -p 80:80 -p 8022:22 --restart always --name gitlab -v C:\installation\gitlab\config:/etc/gitlab -v C:\installation\gitlab\logs:/var/log/gitlab -v C:\installation\gitlab\data:/var/opt/gitlab
```

#### 修改配置

```shell
vi /etc/gitlab/gitlab.rb
-------------------------------------------------
# gitlab访问地址，可以写域名。如果端口不写的话默认为80端口
external_url 'http://gitlab.test'
# ssh主机ip
gitlab_rails['gitlab_ssh_host'] = 'gitlab.test'
# ssh连接端口
gitlab_rails['gitlab_shell_ssh_port'] = 8022


# 是否启用
gitlab_rails['smtp_enable'] = true
# SMTP服务的地址
gitlab_rails['smtp_address'] = "smtp.qq.com"
# 端口
gitlab_rails['smtp_port'] = 465
# 你的QQ邮箱（发送账号）
gitlab_rails['smtp_user_name'] = "*******@qq.com"
# 授权码
gitlab_rails['smtp_password'] = "********"
# 域名
gitlab_rails['smtp_domain'] = "smtp.qq.com"
# 登录验证
gitlab_rails['smtp_authentication'] = "login"
 
# 使用了465端口，就需要配置下面三项
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = true
gitlab_rails['smtp_openssl_verify_mode'] = 'none'
 
# 你的QQ邮箱（发送账号）
gitlab_rails['gitlab_email_from'] = '********@qq.com'
```

```shell
# 修改http和ssh配置
vi /opt/gitlab/embedded/service/gitlab-rails/config/gitlab.yml
--------------------------------------------------------------
gitlab:
  host: gitlab.test
  port: 80 
  https: false
```

```shell
#修改hosts
c:\windows\system32\drivers\etc
127.0.0.1 gitlab.test
```

### gitlab-runner安装

url和token通过gitlab的CI/CD菜单项查看

```shell
docker exec -it gitlab-runner gitlab-runner register -n --url http://172.17.0.2/  --registration-token GR1348941smuNaFuN-Cf68ne2X-tP --executor docker  --description "Docker Runner" --docker-image=maven:latest --tag-list="dev-release"
```

### 注册runner

```shell
docker exec -it gitlab-runner gitlab-runner register -n --url http://172.17.0.2/  --registration-token GR1348941smuNaFuN-Cf68ne2X-tP --executor docker  --description "Docker Runner" --docker-image=maven:latest --tag-list="allen-devolop-tag" 
```

### 项目添加.gitlab-ci.yml文件

```java
# runner使用doker时指定镜像
image: maven:3.6.3-jdk-8

# 定义缓存
# 如果gitlab runner是shell或者docker，此缓存功能没有问题
# 如果是k8s环境，要确保已经设置了分布式文件服务作为缓存
cache:
  key: minio
  paths:
    - .m2/repository/
    - minio/target/*.jar

# 本次构建的阶段：build package
stages:
  - package
  - build

# 生产jar的job
make_jar:
  tags:
    - dev-release
  only:
    - dev-release
  image: maven:3.6.3-jdk-8
  stage: package
  script:
    - echo "=============== 开始编译源码，在target目录生成jar文件 ==============="
    - mvn clean && mvn package -Dmaven.test.skip=true
    - echo "target文件夹" `ls minio/target/`

# 生产镜像的job
make_image:
  tags:
    - dev-release
  only:
    - dev-release
  image: docker:latest
  stage: build
  script:
    - VERSION=`date +%Y%m%d%H%M`
    - echo "VERSION=$VERSION" > .version
    - echo $VERSION
    - echo "从缓存中恢复的target文件夹" `ls minio/target/`
    - echo "=============== 登录Harbor  ==============="
    - cd minio
    - docker login -u user -p password https://harbor.com
    - echo "=============== 打包Docker镜像 ： " harbor.com/test/minio-connect:$VERSION "==============="
    - docker build -t harbor.com/test/minio-connect:$VERSION .
    - echo "=============== 推送到镜像仓库  ==============="
    - docker push harbor.com/test/minio-connect:$VERSION
    - echo "=============== 登出  ==============="
    - docker logout
    - echo "清理掉本次构建的jar文件"
    - rm -rf target/*.jar
```

