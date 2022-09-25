---
title: '前端简单部署'
date: 2022-06-18 11:54:16
tags: [vue,nginx,docker,jenkins]
published: true
hideInList: false
feature: /post-images/KEyilqyTn.png
isTop: false
---
部署前端项目，将打包后的文件dist放入nginx，基于nginx的基础镜像创建前端项目的docker镜像即可

### 一、前端项目的文件准备

#### 1、文件准备：

```shell
1,dockerfile(docker部署镜像打包文件)
2,dist(待部署前端文件夹)
3,default.conf
```

#### 2、Dockerfile文件：

```shell
FROM nginx
RUN rm /etc/nginx/conf.d/default.conf
ADD default.conf /etc/nginx/conf.d/
COPY dist/ /usr/share/nginx/html/
EXPOSE 80
```

#### 3、default.conf：

```shell
server {
    listen       80;
    server_name  localhost; # 修改为docker服务宿主机的ip
  
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        try_files $uri $uri/ /index.html =404;
    }
  
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }
    
    location /api/ {
        proxy_pass http://后端服务名:IP端口;
    }
}
```

------

### 二、vue项目的自动化部署

#### 1、安装Jenkins

> Jenkins的安装：https://tinaxiawuhao.github.io/post/MN_sOyfo3/

#### 2、安装node/npm/yarn

> 安装node与node/npm/yarn：https://tinaxiawuhao.github.io/post/XMNtsw8ig/

#### 3、Jenkins中配置harbor仓库的密钥

我们需要通过jenkinsfile文件推送docker镜像到指定的harbor仓库，需要在jenkins中配置仓库登录凭据

![](https://tianxiawuhao.github.io/post-images/1655524689375.png)



#### 4、编写Jenkinsfile

```
pipeline{
    agent any
 
    environment {
        WS = "${WORKSPACE}"
        HARBOR_URL="harbor的url"
        HARBOR_ID="harbor密钥"
    }
 
    stages {
        stage("环境检查"){
            steps {
                sh 'docker version'
                sh 'npm -v'
                sh 'yarn -v'
            }
        }
        stage('yarn安装与编译') {
            steps {
                sh "echo ${WS} && ls -alh"
                sh "cd ${WS} && yarn install"
                sh "cd ${WS} && yarn build:hd"
            }
        }
        stage('docker镜像与推送') {
            steps {
                sh "cd ${WS} && docker build -f Dockerfile -t ${HARBOR_URL}/test/web:latest ."
                withCredentials([usernamePassword(credentialsId: "${HARBOR_ID}", passwordVariable: 'password', usernameVariable: 'username')]) {
                    sh "docker login -u ${username} -p ${password} ${HARBOR_URL}"
                    sh "docker push ${HARBOR_URL}/test/web:latest"
                }
            }
        }
         
    }
     post {
        success {
            echo 'success!'
        }
        failure {
            echo 'failed...'
        }
    }
}
```

**harbor密钥获取**

![](https://tianxiawuhao.github.io/post-images/1655524703581.png)

点进入刚才添加的凭证获取凭证

![](https://tianxiawuhao.github.io/post-images/1655524712540.png)

最后创建一个pipeline项目（可选择Pipeline或者多分支流水线），pipeline脚本定义选择Pipeline script from SCM，选择git项目

![](https://tianxiawuhao.github.io/post-images/1655524732059.png)

修改关注分支

![](https://tianxiawuhao.github.io/post-images/1655524738660.png)


### 三、模仿后端实现环境变量注入

**借助Nginx1.19以后docker镜像的template新功能，可以实现配置文件环境变量的动态注入**

#### 1、准备 default.conf.template：

```shell
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost; # 修改为docker服务宿主机的ip
 
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        ssi on;
        try_files $uri $uri/ /index.html;
        if ($request_filename ~* ^.*?\.(eot)|(ttf)|(woff)|(svg)|(otf)$) {
            add_header Access-Control-Allow-Origin *;
        }
    }
 
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
 
    location /api/ {
        ssi on;  # 借助ssi
        proxy_pass ${BACKEND_SERVICE_URL};
    }
}
```

#### 2、编写Dockerfile文件：

```shell
FROM nginx:1.21
COPY dist /usr/share/nginx/html
# 注意和上面文件对比位置的改变
COPY default.conf.template /etc/nginx/templates/
EXPOSE 80
```

#### 3、设置环境变量

部署docker时候，设置环境变量-e BACKEND_SERVICE_URL= http://ip:port  后，实际的nginx配置就变为了：

> $BACKEND_SERVICE_URL = http://ip:port #后端服务的地址；

```shell
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost; # 修改为docker服务宿主机的ip
 
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        ssi on;
        try_files $uri $uri/ /index.html;
        if ($request_filename ~* ^.*?\.(eot)|(ttf)|(woff)|(svg)|(otf)$) {
            add_header Access-Control-Allow-Origin *;
        }
    }
 
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
 
    location /api/ {
        ssi on;  # 借助ssi
        proxy_pass  http://ip:port;
    }
}
```

切换服务就可以修改环境变量，切换不同的服务后端