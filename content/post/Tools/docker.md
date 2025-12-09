# docker 学习记录

## 容器管理
```bash
# 查看正在运行的容器
docker ps

# 查看所有容器（包括停止的）
docker ps -a

# 创建并启动一个新容器
docker run -it --name container_name image_name /bin/bash

# 创建并挂载宿主机路径
docker run -it -v /host/path:/container/path image_name /bin/bash
# -e 参数设置环境变量
docker run -it -e http_proxy=http://host.docker.internal:7890 -e https_proxy=http://host.docker.internal:7890 -v /host/path:/container/path image_name /bin/bash


# 进入正在运行的容器
docker exec -it container_name bash

# 停止容器
docker stop container_name

# 启动已停止的容器
docker start container_name

# 重启容器
docker restart container_name

# 删除容器
docker rm container_name

# 删除所有停止的容器
docker container prune

# 将容器保存为镜像
docker commit container_id image_name:tag
docker tag ubuntu2504_dev:last localhost:5000/ubuntu2504_dev:last
docker push localhost:5000/ubuntu2504_dev

# 查看容器日志
docker logs container_name

# 实时跟踪容器日志
docker logs -f container_name
```

## 网络管理
```bash
# 查看网络
docker network ls

# 查看网络详情
docker network inspect network_name

# 创建网络
docker network create my-network

# 删除网络
docker network rm my-network
```

## 卷管理
```bash
# 查看卷
docker volume ls

# 查看卷详情
docker volume inspect volume_name

# 创建卷
docker volume create volume_name

# 删除卷
docker volume rm volume_name

# 删除未使用的卷
docker volume prune
```

## 复制文件
```bash
# 从宿主机拷贝到容器
docker cp /host/path container_name:/container/path

# 从容器拷贝到宿主机
docker cp container_name:/container/path /host/path
```

## 调试和系统命令
```bash
# 查看 Docker 系统信息
docker info

# 查看 Docker 版本
docker version

# 查看容器资源使用情况
docker stats

# 查看容器内部环境变量
docker exec container_name env

# 查看容器内部进程
docker exec container_name ps aux

# 进入容器后查看网络状态
docker exec container_name ifconfig
docker exec container_name ping www.baidu.com
```

## 清理与维护
```bash
# 删除未使用的镜像
docker image prune

# 删除未使用的容器、网络、镜像和卷
docker system prune -a
```