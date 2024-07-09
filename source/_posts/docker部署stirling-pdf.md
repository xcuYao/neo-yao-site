---
title: docker部署stirling_pdf
date: 2024-07-09 15:40:51
tags:
---

### debian安装docker
#### 升级包
``` bash
sudo apt update && sudo apt upgrade -y
```

#### 加 Docker 官方 GPG key
``` bash
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/debian/gpg | sudo apt-key add -
``` 

#### 写入软件源信息
``` bash
sudo add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/debian $(lsb_release -cs) stable"
``` 

#### 更新软件包列表
``` bash
sudo apt-get -y update
``` 

#### 安装
``` bash
sudo apt-get -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
``` 

#### 查看
``` bash
docker version
docker compose version
``` 

### dock修改镜像源
#### 编辑配置文件
``` bash
sudo nano /etc/docker/daemon.json
```
录入如下信息
``` json
{
    "registry-mirrors": [
    "https://ghcr.io",
    "https://5yt01a9i.mirror.aliyuncs.com",
    "https://registry.docker-cn.com",
    "http://hub-mirror.c.163.com",
    "https://mirror.baidubce.com"
    ]
}

```

#### 刷新docker源
``` bash
sudo systemctl daemon-reload 
sudo systemctl restart docker
```

#### 查看
``` bash
sudo docker info
```

### 部署 Stirling PDF

#### 创建安装目录
``` bash
sudo -i
mkdir -p /root/data/docker_data/stirling_pdf
cd /root/data/docker_data/stirling_pdf
```

#### 创建并编辑 docker-compose.yml文件 

``` bash
vim docker-compose.yml
```

``` yaml
version: '3.3'
services:
  stirling-pdf:
    image: frooodle/s-pdf:latest
    ports:
      - '8080:8080'
    volumes:
      - ./trainingData:/usr/share/tessdata #Required for extra OCR languages
      - ./extraConfigs:/configs
#      - ./customFiles:/customFiles/
#      - ./logs:/logs/
    environment:
      - DOCKER_ENABLE_SECURITY=false
      - INSTALL_BOOK_AND_ADVANCED_HTML_OPS=false
```

#### 打开防火墙的8080端口
云服务器控制台-防火墙操作

#### 查看端口是否占用
``` bash
// 如果啥也没出现，表示端口未被占用
lsof -i:8080
```

#### 启动 stirling_pdf
``` bash
cd /root/data/docker_data/stirling_pdf
docker compose up -d
```

#### 检查结果
输入http://ip:8080 即可访问
