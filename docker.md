> [Docker: Get Started](docs.docker.com/get-started)

## 1. Container

##### Dockerfile

```dockerfile
# Use an official Python runtime as a parent image
FROM python:2.7-slim

# Set the working directory to /app
WORKDIR /app

# Copy the current directory contents into the container at /app
ADD . /app

# Install any needed packages specified in requirements.txt
RUN pip install --trusted-host pypi.python.org -r requirements.txt

# Make port 80 available to the world outside this container
EXPOSE 80

# Define environment variable
ENV NAME World

# Run app.py when the container launches
CMD ["python", "app.py"]
```

##### Build

```sh
docker build -t friendlyhello .
```

`-t` 指定编译后的 image 名称。

##### Run

```sh
docker run -p 4000:80 friendlyhello
```

`-p` 将 host 的 4000 端口映射到 image 里的 80 端口。

##### Tag

```sh
docker tag image_name username/repository:tag
```

例如：

```sh
docker tag friendlyhello tomisacat/get-started:part2
```

##### Publish

将你创建的 image 上传到 docker cloud：

```sh
docker push tomisacat/get-started:part2
```

##### Pull and run

将 docker cloud 上的 image 拉取到本地并运行：

```sh
docker run -p 4000:80 tomisacat/get-started:v1
```

## 2. Service

Service 就是生产环境下的 Container，每个 service 只运行一个 image，它会编排这个 image 如何运行，例如使用哪个端口，复制多少个 image 运行实例。这些操作都通过 `docker-compose.yml` 文件来定义。

##### docker-compose.yml

```yml
version: "3"
services:
  # service name is "web"
  web:
    # replace username/repo:tag with your name and image details
    image: username/repo:tag
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
      restart_policy:
        condition: on-failure
    ports:
      - "4000:80"
    networks:
      - webnet
networks:
    webnet:
```

##### Run

初始化容器编排

```sh
docker swarm init
```

运行 service：

```sh
docker stack deploy -c docker-compse.yml getstartedlab
```

查看一下：

```sh
$ docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                        PORTS
d62p7wjzanu2        getstartedlab_web   replicated          5/5                 tomisacat/friendlyhello:v1   *:4000->80/tcp
```

service 里跑的每个 container 都被称为一个 task，查看：

```sh
$ docker service ps getstartedlab_web
ID                  NAME                  IMAGE                        NODE                    DESIRED STATE       CURRENT STATE         ERROR               PORTS
aetdgojijmfk        getstartedlab_web.1   tomisacat/friendlyhello:v1   linuxkit-025000000001   Running             Running 7 hours ago
8l2gkpilhkjr        getstartedlab_web.2   tomisacat/friendlyhello:v1   linuxkit-025000000001   Running             Running 7 hours ago
tyewjqegj0ot        getstartedlab_web.3   tomisacat/friendlyhello:v1   linuxkit-025000000001   Running             Running 7 hours ago
y2c7cyj2ptcv        getstartedlab_web.4   tomisacat/friendlyhello:v1   linuxkit-025000000001   Running             Running 7 hours ago
29m4ev1ca788        getstartedlab_web.5   tomisacat/friendlyhello:v1   linuxkit-025000000001   Running             Running 7 hours ago
```

##### Take down

停止运行 app（service）：

```sh
docker stack rm getstartedlab
```

停止 Swarm：

```sh
docker swarm leave --force
```

## 3. Swarm

一组跑 docker 服务的机器（machines）组成的集群称为 Swarm。机器可以是 virtual 或 physical 的，加入到 swarm 里后称为 `nodes`。执行 `docker swarm init` 指令可以将当前机器初始化为一台 swarm 管理器。

##### Create a cluster

```sh
docker-machine create --driver virtualbox myvm1
docker-machine create --driver virtualbox myvm2
```

##### List VMs

```sh
$ docker-machine ls
NAME    ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
myvm1   -        virtualbox   Running   tcp://192.168.99.100:2376           v17.06.2-ce
myvm2   -        virtualbox   Running   tcp://192.168.99.101:2376           v17.06.2-ce
```

##### Initialize

可以通过 `docker-machine ssh` 向 VM 发送指令。例如，将 myvm1 初始化为 swarm 管理器：

```sh
$ docker-machine ssh myvm1 "docker swarm init --advertise-addr <myvm1 ip>"
Swarm initialized: current node <node ID> is now a manager.

To add a worker to this swarm, run the following command:

  docker swarm join \
  --token <token> \
  <myvm ip>:<port>

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

> 上面的指令里有设置端口号的参数，但最好固定使用 2377 或者不写（使用默认的端口号）

查看当前 swarm 有多少 nodes：

```sh
$ docker-machine ssh myvm1 "docker node ls"
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
brtu9urxwfd5j0zrmkubhpkbd     myvm2               Ready               Active
rihwohkh3ph38fhillhhb84sk *   myvm1               Ready               Active              Leader
```

> 如果想要删除某个 node，直接向那台 vm 发送 `docker swarm leave`（通过 docker-machine ssh）就可以了

##### Deploy

执行下面指令来获取与 swarm 管理器交互的配置：

```sh
$ docker-machine env myvm1
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://192.168.99.100:2376"
export DOCKER_CERT_PATH="/Users/sam/.docker/machine/machines/myvm1"
export DOCKER_MACHINE_NAME="myvm1"
# Run this command to configure your shell:
# eval $(docker-machine env myvm1)
```

所以，可以直接调用：

```sh
eval $(docker-machine env myvm1)
```

调用 `docker-machine ls` 查看是否已经将 myvm1 设置为活动机器（active machine），也就是用 * 标记的：

```sh
$ docker-machine ls
NAME    ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
myvm1   *        virtualbox   Running   tcp://192.168.99.100:2376           v17.06.2-ce
myvm2   -        virtualbox   Running   tcp://192.168.99.101:2376           v17.06.2-ce
```

此时，你可以直接执行指令，这些指令将在 myvm1 上生效：

```sh
docker stack deploy -c docker-compose.yml getstartedlab
```

> 如果 image 文件在私有服务器上则需要 `docker login  <yout-registry>` 登陆并添加 flag：
> ```sh
> docker login registry.example.com
> docker stack deploy --with-registry-auth -c docker-compose.yml getstartedlab
> ```

##### Cleanup & reboot

像之前一样，关闭服务：

```sh
docker stack rm getstartedlab
```

重置（清除） shell 变量：

```sh
eval $(docker-machine env -u)
```

重启一台 machine：

```sh
docker-machine start <machine-name>
```

## 4. Stacks

##### Add new service

编辑 docker-compose.yml：

```yml
version: "3"
services:
  web:
    # replace username/repo:tag with your name and image details
    image: username/repo:tag
    deploy:
      replicas: 5
      restart_policy:
        condition: on-failure
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
    ports:
      - "80:80"
    networks:
      - webnet
  visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints: [node.role == manager]
    networks:
      - webnet
networks:
  webnet:
```

新增了一个 `visualizer` 服务，并且有两个新的参数：`volumes`，允许 visualizer 访问 host 的 socket 文件；`placement`，这里设置 visualizer 服务只（constraints）运行在 swarm 管理器而不能运行在一个（受 swarm 管理器管理的）worker 上。

##### Deploy

```sh
$ docker stack deploy -c docker-compose.yml getstartedlab
Updating service getstartedlab_web (id: angi1bf5e4to03qu9f93trnxm)
Creating service getstartedlab_visualizer (id: l9mnwkeq2jiononb5ihz9u7a4)
```

可以打开浏览器 `http://192.168.99.101:8080` 看效果。

##### Dependency

在 docker-compose.yml 文件里添加如下服务：

```yml
...

redis:
    image: redis
    ports:
      - "6379:6379"
    volumes:
      - "/home/docker/data:/data"
    deploy:
      placement:
        constraints: [node.role == manager]
    command: redis-server --appendonly yes
    networks:
      - webnet
      
...
```

这里指定 redis 可以访问 host 上的 `./data` 并映射为 container 里的 `/data`，因此需要先创建一个 `./data` 目录：

```sh
docker-machine ssh myvm1 "mkdir ./data"
```

并根据上面的操作将 myvm1 设置为 active machine：

* 执行 `docker-machine ls` 检查是否已经将 myvm1 设置为 active machine
* 如果没有，执行 `docker-machine env myvm1`，并 `eval $(docke-machine env myvm1)`

接着执行：

```sh
docker stack deploy -c docker-compose.yml getstartedlab
```

检查是否有三个服务正在运行：

```sh
$ docker service ls
ID                  NAME                       MODE                REPLICAS            IMAGE                             PORTS
x7uij6xb4foj        getstartedlab_redis        replicated          1/1                 redis:latest                      *:6379->6379/tcp
n5rvhm52ykq7        getstartedlab_visualizer   replicated          1/1                 dockersamples/visualizer:stable   *:8080->8080/tcp
mifd433bti1d        getstartedlab_web          replicated          5/5                 gordon/getstarted:latest    *:80->80/tcp
```


