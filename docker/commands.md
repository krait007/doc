# Docker 常用命令简介 

                          ##         .
                     ## ## ##        ==
                  ## ## ## ## ##    ===
              /"""""""""""""""""\___/ ===
         ~~~ {~~ ~~~~ ~~~ ~~~~ ~~~ ~ /  ===- ~~~
              \______ o __/ \\ __/
               \____\_______/
    _ ______ _
    | |__   ___   ___ | |_|___ \ __| | ___   ___| | _____ _ __
    | '_ \ / _ \ / _ \| __| __) / _` |/ _ \ / __| |/ / _ \ '__|
    | |_) | (_) | (_) | |_ / __/ (_| | (_) | (__|   <  __/ |
    |_.__/ \___/ \___/ \__|_____\__,_|\___/ \___|_|\_\___|_|
    


## `docker` 命令
<pre><code>
    Usage:	docker COMMAND

    A self-sufficient runtime for containers

    Options:
        --config string      Location of client config files (default "/Users/krait/.docker")
    -D, --debug              Enable debug mode
        --help               Print usage
    -H, --host list          Daemon socket(s) to connect to
    -l, --log-level string   Set the logging level ("debug"|"info"|"warn"|"error"|"fatal") (default "info")
        --tls                Use TLS; implied by --tlsverify
        --tlscacert string   Trust certs signed only by this CA (default "/Users/krait/.docker/ca.pem")
        --tlscert string     Path to TLS certificate file (default "/Users/krait/.docker/cert.pem")
        --tlskey string      Path to TLS key file (default "/Users/krait/.docker/key.pem")
        --tlsverify          Use TLS and verify the remote
    -v, --version            Print version information and quit

    Management Commands:
    config      Manage Docker configs
    container   Manage containers
    image       Manage images
    network     Manage networks
    ...

    Commands:
    attach      Attach local standard input, output, and error streams to a running container
    build       Build an image from a Dockerfile
    commit      Create a new image from a container's changes
    cp          Copy files/folders between a container and the local filesystem
    . . .
    Run 'docker COMMAND --help' for more information on a command.
</code></pre>

***  
### `docker run` 运行一个容器  
#### `docker run alpine echo Hello World!`    
  运行镜像alpline一个容器，启动后执行echo 命令.  
  <pre><code>
    Unable to find image 'alpine:latest' locally
    latest: Pulling from library/alpine
    b56ae66c2937: Pull complete 
    Digest: sha256:b40e202395eaec699f2d0c5e01e6d6cb8e6b57d77c0e0221600cf0b5940cf3ab
    Status: Downloaded newer image for alpine:latest
    Hello World!
  </code></pre>

#### `docker run -t -i alpine /bin/sh`  
  运行镜像alpline一个容器，启动后执行sh, -it进入交互模式
  <pre><code>
/ # ls
bin    dev    etc    home   lib    media  mnt    proc   root   run    sbin   srv    sys    tmp    usr    var
/ # exit
  </code></pre>


### `docker run -it -d alpine top`   
  后台模式运行容器 -d  
  <pre><code>
  8c52d0b9abb892de0c4a763784f91cd2876d4a9bfe6eaadc9ae65cc761687416
  </code></pre>


### `docker run -it -p 16789:6789 alpine /usr/bin/nc -l 6789`  
  启动容器并运行nc监听 6789端口， host主机上映射到16789  
  <pre><code>
  telnet localhost 16789
  Trying ::1...
  Connected to localhost.
  Escape character is '^]'.
  Connection closed by foreign host.
  </code></pre>

### `docker run --help`   
  获取run命令的详细  
  -e "SERVICE_PORT=1111" 传入容器的环境变量  
  -h hostname  指定容器的hostname  
  --name 指定容器的名称，不指定系统默认随机生成  
  --rm 容器运行完退出后自动删除  
  --add-host 添加ip和主机名的映射关系到容器里的/etc/hosts文件  
  -v /host:/container 挂载本地存储到容器中  
  --net=host 指定网络命名空间namespace  
   
  <pre><code>
  Usage:	docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

  Run a command in a new container

  Options:
      --add-host list                  Add a custom host-to-IP mapping (host:ip)
  -a, --attach list                    Attach to STDIN, STDOUT or STDERR
      --blkio-weight uint16            Block IO (relative weight), between 10 and 1000, or 0 to disable (default 0)
      --blkio-weight-device list       Block IO weight (relative device weight) (default [])
      --cap-add list                   Add Linux capabilities
        ...
  </code></pre>

### 容器生命周期： 创建, 启动, 关闭, 删除  
#### `docker create -p 16789:6789 alpine /usr/bin/nc -l 6789`  
  创建容器  
  <pre><code>
  acceadd502ebddf20231e98bdb9dcbbbeec85732a2b200818625dcee5caa3799
  
  CONTAINER ID        IMAGE               COMMAND                 CREATED                  STATUS              PORTS               NAMES
  acceadd502eb        alpine              "/usr/bin/nc -l 6789"   Less than a second ago   Created                                 stupefied_hodgkin
  </code></pre>

 #### `docker ps -a`  
  查看当前所有容器  
  <pre><code>
  CONTAINER ID        IMAGE               COMMAND                 CREATED                  STATUS              PORTS               NAMES
  acceadd502eb        alpine              "/usr/bin/nc -l 6789"   Less than a second ago   Created                                 stupefied_hodgkin
  </code></pre> 

#### `docker start acceadd502eb`  
  启动容器 acceadd502eb  
  
#### `docker ps`  
  查看当前所有容器  
  <pre><code>
  CONTAINER ID        IMAGE               COMMAND                 CREATED             STATUS              PORTS                     NAMES
acceadd502eb        alpine              "/usr/bin/nc -l 6789"   4 minutes ago       Up 44 seconds       0.0.0.0:16789->6789/tcp   stupefied_hodgkin
  </code></pre> 

#### `docker exec acceadd502eb ls`  
  容器执行命令  
    <pre><code>
        bin
        dev
        etc
        home
        lib
        media
        mnt
        proc
        root
        run
        sbin
        srv
        sys
        tmp
        usr
        var
  </code></pre> 


#### `docker kill acceadd502eb | docker stop acceadd502eb`
  关闭容器
  
#### `docker rm acceadd502eb `
  删除容器
  
***  
### `docker images`   
  docker镜像  
  <pre><code>
    REPOSITORY                            TAG                 IMAGE ID            CREATED             SIZE
    alpine                                latest              37eec16f1872        3 days ago          3.97MB
  </code></pre> 

### 获取创建镜像文件  
- pull镜像文件到本地  
  `docker pull alpine`  
  获取镜像到本机，默认从 hub.docker.com上下载，可以设置代理地址，或者指定地址pull  
   `docker pull gcr.io/google_containers/kube-cross`  

-  运行本地镜像，登录容器运行操作命令，然后commit  保存镜像  
  <pre><code>
  docker run -d -it alpine top
  eefc9afeebbb2ce4db6d4647b79800104cdf71cf5b3aa963409a521a292c04c7

  docker exec -it eefc9afeebbb sh
  / # echo hello >/hello.txt
  / # exit
  
  docker commit  eefc9afeebbb alpine:demo
  sha256:5f86c37be3801d1ef113138cd902c4b970c2dcc49ef5715a590e5fe9b11aa650

  docker images
  REPOSITORY                            TAG                 IMAGE ID            CREATED             SIZE
  alpine                                demo                5f86c37be380        27 seconds ago      3.97MB
  alpine                                latest              37eec16f1872        3 days ago          3.97MB

  docker diff eefc9afeebbb
  A /hello.txt
  C /root
  A /root/.ash_history
  </code></pre> 


- 导出导入镜像  
    `docker save alpine:latest > alpine.tar`   
    `docker load < alpine.tar` load | import  
  
- Build定制镜像  
  Dockerfile   
  <pre><code>
  # vi Dockerfile
  
  FROM alpine:latest
  MAINTAINER jinjianhong vcfan@yeah.net
  
  # install nginx
  RUN apk --update add nginx
  
  EXPOSE 80
  
  CMD ["nginx", "-g", "daemon off;"]
  </code></pre> 
  
  编译镜像
  `docker build -t nginx:demo .`
  <pre><code>
    Sending build context to Docker daemon  33.28kB
    Step 1/5 : FROM alpine:latest
    ---> 37eec16f1872
    Step 2/5 : MAINTAINER jinjianhong vcfan@yeah.net
    ---> Running in 395ed31913b7
    ---> 1db069ead6bc
    Removing intermediate container 395ed31913b7
    Step 3/5 : RUN apk --update add nginx
    ---> Running in 90efb8af98a3
    fetch http://dl-cdn.alpinelinux.org/alpine/v3.6/main/x86_64/APKINDEX.tar.gz
    fetch http://dl-cdn.alpinelinux.org/alpine/v3.6/community/x86_64/APKINDEX.tar.gz
    (1/2) Installing pcre (8.41-r0)
    (2/2) Installing nginx (1.12.2-r0)
    Executing nginx-1.12.2-r0.pre-install
    Executing busybox-1.26.2-r7.trigger
    OK: 5 MiB in 13 packages
    ---> 147c3c80c6d0
    Removing intermediate container 90efb8af98a3
    Step 4/5 : EXPOSE 80
    ---> Running in 68c7ddef6ec9
    ---> f354acccf0a9
    Removing intermediate container 68c7ddef6ec9
    Step 5/5 : CMD nginx -g daemon off;
    ---> Running in 865019c62890
    ---> 0b9e608a5ad3
    Removing intermediate container 865019c62890
    Successfully built 0b9e608a5ad3
    Successfully tagged nginx:demo
  </code></pre> 

  `docker images`
  <pre><code>
    REPOSITORY                            TAG                 IMAGE ID            CREATED             SIZE
    nginx                                 demo                0b9e608a5ad3        4 minutes ago       6.43MB
    alpine                                demo                5f86c37be380        19 minutes ago      3.97MB
    alpine                                latest              37eec16f1872        3 days ago          3.97MB
  </code></pre> 
  
### 镜像常用操作  
  镜像更新tag  
  `docker tag nginx:demo nginx:tag`  
  删除镜像  
  `docker rmi nginx:tag`  
  `docker rmi nginx:tag -f` 当存在容器时先删除容器或强制删除-f  
  
  上传镜像，需要权限  
  `docker push nginx:demo`   

  搜索镜像  
  `docker search postgres`  
  <pre><code>
    NAME                                         DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
    postgres                                     The PostgreSQL object-relational database ...   4141                [OK]                
    sameersbn/postgresql                                                                         116                                     [OK]
    paintedfox/postgresql                        A docker image for running Postgresql.          76                                      [OK]
    orchardup/postgresql                         https://github.com/orchardup/docker-postgr...   36                                      [OK]
    kiasaki/alpine-postgres                      PostgreSQL docker image based on Alpine Linux   35                                      [OK]
                              [OK]
    ...
  </code></pre> 
    

### docker对象信息查看
  docker inspect [OPTIONS] NAME|ID [NAME|ID...]  
  - 获取镜像信息  
    `docker inspect alpine`  

  - 获取容器信息  
    `docker inspect eefc9afeebbb`

  - 使用 -f 获取指定项目的值   
    获取IP地址  
    `docker inspect -f '{{ .NetworkSettings.IPAddress}}' eefc9afeebbb`  
    获取容器状态  
    `docker inspect -f '{{ .State.Status}}' eefc9afeebbb`
    
