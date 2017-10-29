# Docker搭建私有Registry

## docker compose 配置文件
  - 使用官方的镜像 registry
    registry的image 保存在/tmp/registry中, 挂载本机volume持久保存
    docker-compose.yml
    <pre><code>   
    registry:
        image: registry
        ports:
          - "5000:5000"
        volumes:
          - /opt/data/registry:/tmp/registry
        environment:
          - STORAGE_PATH:/tmp/registry
          - SETTINGS_FLAVOR:dev
    </code></pre>

  - nginx配置文件
    将本机IP映射成域名
    注意proxy_pass不带端口号
    default.conf
    <pre><code>   
    # registry
    server {
        listen       80 ;
        server_name docker.bdbyod.com;
        location / {
            proxy_pass http://$localeIP;
        }
    }
    </code></pre>

  - docker 服务配置文件  
    centos在 /etc/sysconfig/docker  
    ubuntu在 /etc/default/docker  
    因为默认是只允许https上传下载，所以要指定安全地址  
    
    重启docker服务  
    <pre><code>  
      other_args="--insecure-registry docker.bdbyod.com:5000"
    </code></pre>

    打好tag的需要上传的镜像  
    docker tag bdbyod/nginx docker.bdbyod.com:5000/bdbyod/nginx:3.0

  - 开始运行
  <pre><code>  
  docker-compose up -d registry
  service nginx start
  curl -i docker.bdbyod.com:5000
  curl -i http://docker.bdbyod.com:5000/v1/search
  docker push docker.bdbyod.com:5000/bdbyod/nginx:3.0
  ls /opt/data/registry/ //检查镜像是否存在
  docker pull docker.bdbyod.com:5000/bdbyod/nginx:3.0
  </code></pre>

