# Ubuntu_sever_setup

Docker: Sử dụng Mongodb, Nginx
Node, React app chạy trên nền PM2

**Khái niệm**
- Docker là công cụ giúp thực hiện việc thiết lập và cài đặt môi trường cho app nhanh chóng.
- Dùng lệnh để cài đặt dependencies từ dockerhub.
- Sử dụng Dockerfile để tự động tạo môi trường như cài đặt, tạo thư mục.
- Docker-compose thực hiện lệnh chạy một lần cho các service database, backend, frontend

*Ghi chú*: để thuận tiện thì nên `ssh` server trên máy local.

## Thiết kế cấu trúc cần cài đặt

```
srv/
|
├─ PM2/
│ └─ Dockerfile
|
├─ frontend/
│ ├─ build
│ └─ Dockerfile
│
├─ api/
│ ├─ build
│ └─ Dockerfile
│
└─ docker-compose.yml
```

## Cài đặt môi trường ban đầu
- Cài Ubuntu (lts 16) trên server.
- Cài Docker.
- Cài npm

## Cài Docker compose
Cài đặt version docker compose mới nhất (hiện tại là 1.17.1)
`sudo curl -o /usr/local/bin/docker-compose -L "https://github.com/docker/compose/releases/download/1.17.1/docker-compose-$(uname -s)-$(uname -m)"`

Cấu hình permission cho ubuntu.
`sudo chmod +x /usr/local/bin/docker-compose`

Kiểm tra phiên bản
`docker-compose -v`


## Dockerfile để deploy static React app

React app chạy trên service api nên chỉ cần deploy static

Tạo file Dockerfile cho container frontend như sau
* Sử dụng serve để chạy react app https://github.com/zeit/next.js/

```Dockerfile
# Based image from node
FROM node:8
#The base node image sets a very verbose log level.
ENV NPM_CONFIG_LOGLEVEL warn
# Copy all local files into the image.
COPY . .
# Install serve
RUN npm install -g serve
# Create app directory
RUN mkdir -p /srv/www/frontend
# Set working dir
WORKDIR /srv/www/frontend
# Command to run
CMD serve -s /srv/www/frontend
# Tell docker the port
EXPOSE 3000
```
Truy cập thư mục `srv/frontend`

Chạy `docker build ./` để test build
Sau đó chạy `docker run -i -t -p 3000:5000 [built-ID]` chạy cái build id

Ok rồi thì có thể stop & remove cái container đó đi (kiểm tra container `docker ps -a`)

## Cài Nginx
Cài đặt Nginx để tạo load React chạy trên domain

Pull nginx từ docker hub `docker pull nginx:latest`
Tạo Dockerfile

```Dockerfile
# Set nginx base image
FROM nginx
# Remove existed config file
RUN rm -rf /etc/nginx/nginx.conf
# Copy custom configuration file from the current directory
COPY nginx.conf /etc/nginx/nginx.conf
# Append "daemon off;" to the beginning of the configuration
# in order to avoid an exit of the container
RUN echo "daemon off;" >> /etc/nginx/nginx.conf
# Expose ports
EXPOSE 443
# Define default command
CMD service nginx start
```
Tạo file nginx.conf

`nano nginx.conf`

```conf
worker_processes  1;

events {
  worker_connections 1024;
}

http {
  proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=one:10m;
  proxy_temp_path /var/tmp;

  gzip on;
  gzip_comp_level 4;
  gzip_min_length 500;

	##################################
	# REACT
	###################################
	server {
		listen 80;
    charset utf-8;
		root /srv/www/frontend/build;
		server_name my_domain.com;

		# Routes without file extension e.g. /user/1
		location / {
			try_files $uri /index.html;
			add_header   Cache-Control public;
		}

		# [optional]404 if a file is requested (so the main app isn't served)
		location ~ ^.+\..+$ {
			try_files $uri =404;
		}

		# [optional]redirect server error pages to the static page /50x.html
		# error_page   500 502 503 504  /50x.html;
		# location = /50x.html {
		# 	root   /usr/share/nginx/html;
		# }
	}

}
```

Cấu hình nginx: sẽ chạy trên port web 80, file `index.html` lại thư mục /srv/www/frontend

Soạn file `docker-compose.yml` như sau:

```yml
version: '3'
services:
  #####################
  # NGINX
  #####################
  nginx:
    image         : nginx:stable
    container_name: Nginx
    # Dockerfile location
    build         : ./nginx
    links         :
      - react
    volumes       :
      - ./nginx/:/etc/nginx/
    ports:
      - 8080:80
      - 443:443

  #####################
  # REACT
  #####################
  react:
    container_name: React
    # Dockerfile location
    build         : ./frontend
    volumes       :
      - .:/srv/www/frontend
    ports         :
      - 3000:5000
```

Chạy build test `docker-compose up --build -d`


## Cài mongodb

```t
mkdir ~/database
sudo docker run -d -p 27017:27017 -v ~/database:/data/db mongo
```
có thể đặt Port khác `CUSTOM_PORT:27017` nếu cần

Kiểm tra danh sách docker containers `sudo docker ps`

Do tên container tự generate nên có thể đổi tên bằng cách `sudo docker rename old-name database`

Chạy mongodb auth `sudo docker run -d --name database -p 27017:27017 -v ~/database:/data/db mongo --auth`

`docker stop database`
`docker rm database`
nếu bị conflict

Truy cập mongo `docker exec -it database mongo admin`

Tạo tài khoản root `db.createUser({ user: 'admin', pwd: 'password', roles: [ { role: "root", db:"admin" } ] });`

Thoát mongo bằng lệnh `exit`

Chạy mongo theo tài khoản đã thực hiện `docker exec -it database mongo -u "admin" -p "password" --authenticationDatabase "admin"`
Tạo tài khoản cho db

```js
use mydatabase
db.createUser ({
  user: "admin",
  pwd: "password",
  roles: [ { role: "root", db: "admin" } ]
});

```

Dừng chạy database container `docker stop database`, remove container `docker rm database` (phải remove nếu không sẽ bị conflict).

## Cài PM2 và chạy node api

```t
mkdir nodeapp
nano Dockerfile
```
Watch node app running với PM2
Tạo file
```Dockerfile
# Set the base image to Ubuntu
FROM ubuntu:latest

# clean and update sources
RUN apt-get -y clean && apt-get -y update && apt-get install --assume-yes apt-utils

# Install Node.js and other dependencies
RUN apt-get -y install curl && \
    apt-get -y install git && \
    apt-get -y install wget && \
    curl -sL https://deb.nodesource.com/setup_8.x | bash - && \
    apt-get -y install nodejs

# Install PM2
RUN npm install -g pm2

RUN mkdir -p /srv/www/api

# Define working directory
WORKDIR /srv/www/api

ADD . /srv/www/api

COPY docker-entrypoint.sh /
ENTRYPOINT ["/docker-entrypoint.sh"]

# Expose port
EXPOSE 5000

# Run app
CMD pm2 start --no-daemon processes.json
```
Nhớ copy cả cái entrypoint

processes.json
```json
{
  "apps" : [{
    "merge_logs"  : true,
    "name"        : "api",
    "out_file"    : "/tmp/servers.log",
    "log_date_format" : "MM/DD/YYYY HH:mm:ss",
    "script"      : "srv/www/api/index.js"
  },{
    "merge_logs"  : true,
    "name"        : "pm2-notifier",
    "out_file"    : "/tmp/pm2-notifier.log",
    "script"      : "lib/pm2-notifier.js",
    "env": {
      "EC2": "ENV_CTXT"
    }
  }]
}
```

Hoàn tất, chạy lệnh `docker-compose up --build -d services`


stop, kill all containers
kill images
```
docker stop $(docker ps -a -q)
docker rm $(docker ps -a -q)
docker rmi $(docker images -q)
```
[more](https://www.digitalocean.com/community/tutorials/how-to-remove-docker-images-containers-and-volumes)