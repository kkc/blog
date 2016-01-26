title: docker registry 的簡單安裝紀錄
date: 2016-01-14 06:12:08
tags: docker
---

![docker_registry](/img/docker_registry.png)

## 前言 ##

最近公司也採用自己搭建 docker registry，完成 CI/CD 的最後一厘路，這邊做個簡單的安裝紀錄。

## 安裝 ##

### 安裝 docker registry ###

先把 image 從 dockerhub 上面撈下來，我是用 2.1 這個 version，不過官方版本好像到 2.2.1 了。
```
docker pull registry:2.1
```

先建立好 config，然後主要是要把 images 都推上 s3，記得要先去建立好相對應的 IAM user，建立好 accesskey 和 secretkey，其實我會更推薦使用 IAM role 搭配 EC2 ，這樣 accesskey 和 secretkey 就可以留下 ""
```
version: 0.1
log:
  level: debug
  formatter: text
  fields:
    service: registry
    environment: staging
  hooks:
    - type: mail
      disabled: true
      levels:
        - panic
      options:
        smtp:
          addr: mail.example.com:25
          username: mailuser
          password: password
          insecure: true
        from: sender@example.com
        to:
          - errors@example.com
loglevel: debug # deprecated: use "log"
storage:
  cache:
    blobdescriptor: inmemory
  s3:
    accesskey: ""
    secretkey: ""
    region: your-region-name
    bucket: your-bucket-name
    encrypt: true
    secure: true
    v4auth: true
    chunksize: 5242880
    rootdirectory: /
  redirect:
    disable: true
http:
    addr: :5000
    headers:
        X-Content-Type-Options: [nosniff]
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
```

啟動
```
docker run -d -p 5000:5000 --restart=always --name registry -v `pwd`/config.yml:/etc/docker/registry/config.yml registry:2.1
```

測試一下 registry 是否有正常運作，首先把已經有的 image 打上 tag，在試著推上去撈下來
```
docker tag ubuntu localhost:5000/ubuntu
docker pull localhost:5000/ubuntu
docker push localhost:5000/ubuntu
```

### 安裝 nginx ###

這邊要用 nginx 安裝 reverse proxy，主要是因為我們想要走 https 加密的方式，加上 nginx 也可以 serve 簡單的認證功能，不過 nginx 的版本要高一點，要不然無法使用 `add_header`。

```
sudo add-apt-repository ppa:nginx/stable
sudo apt-get install nginx
```

編輯 /etc/nginx/sites-enabled/registry
```
upstream docker-registry {
  server 127.0.0.1:5000;
}

server {
  listen 443 ssl;
  server_name registry.your_domain.com;

  # SSL
  ssl_certificate /etc/nginx/bundle.crt;
  ssl_certificate_key /etc/nginx/certificate.key;

  # disable any limits to avoid HTTP 413 for large image uploads
  client_max_body_size 0;

  # required to avoid HTTP 411: see Issue #1486 (https://github.com/docker/docker/issues/1486)
  chunked_transfer_encoding on;

  location /v2/ {
    # Do not allow connections from docker 1.5 and earlier
    # docker pre-1.6.0 did not properly set the user agent on ping, catch "Go *" user agents
    if ($http_user_agent ~ "^(docker\/1\.(3|4|5(?!\.[0-9]-dev))|Go ).*\$" ) {
      return 404;
    }

    # To add basic authentication to v2 use auth_basic setting plus add_header
    auth_basic "Registry realm";
    auth_basic_user_file /etc/nginx/htpasswd;
    add_header 'Docker-Distribution-Api-Version' 'registry/2.0' always;

    proxy_pass                          http://docker-registry;
    proxy_set_header  Host              $http_host;   # required for docker client's sake
    proxy_set_header  X-Real-IP         $remote_addr; # pass on real client's IP
    proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header  X-Forwarded-Proto $scheme;
    proxy_read_timeout                  900;
  }
}
```

產生 /etc/nginx/htpasswd，建立 user & password ，然後重新啟動 nginx
```
sudo htpasswd -c /etc/nginx/htpasswd exampleuser
sudo servie nginx restart
```

驗證登入
```
docker login your_registry.domain_name.com
```
應該就會看到要求輸入帳號密碼

## trouble shooting ##

可以看到 ssl 是不是有問題
```
curl -i -k -v https://account:password@your_registry.domain_name.com/v2/
```

檢查 push 上面的 images 有沒有存在
```
curl -i -k -v https://account:password@your_registry.domain_name.com/v2/_catalog
{"repositories":["ubuntu"]}
```
