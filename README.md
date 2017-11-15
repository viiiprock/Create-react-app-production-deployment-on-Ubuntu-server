# Ubuntu_sever_setup

Docker: Sử dụng Mongodb, Nginx

Node, React app chạy trên nền PM2

Đây là bài viết để lưu trữ, khi cần lôi ra coi cho nhanh.

**Khái niệm**
- Docker là công cụ giúp thực hiện việc thiết lập và cài đặt môi trường cho app nhanh chóng.
- Dùng lệnh để cài đặt dependencies từ dockerhub.
- Sử dụng Dockerfile để tự động tạo môi trường như cài đặt, tạo thư mục.
- Docker-compose thực hiện tổng thể cho các service database, backend, frontend

*Ghi chú*: để thuận tiện thì nên `ssh` server trên máy local.

## Thiết kế cấu trúc cần cài đặt

Tùy thuộc vào yêu cầu mà thiết kế cấu trúc cài đặt cho phù hợp với dự án.


```
srv/
├─ frontend/
│ ├─ build
│ └─ Dockerfile
│
└─ api/
	├─ build
  └─ Dockerfile

```

## Cài đặt môi trường ban đầu
- Cài Ubuntu (lts 16)
- Cài Docker.
- Cài npm

## Dockerfile để deploy static React app

React app chạy trên service api nên chỉ cần deploy static

Tạo file Dockerfile cho container frontend như sau

```Dockerfile
# Based image from node
FROM node:8
# The base node image sets a very verbose log level.
ENV NPM_CONFIG_LOGLEVEL warn
# Copy all local files into the image.
COPY . .
# Install serve
RUN npm install -g serve
# Create app directory
RUN mkdir -p /var/frontend
# Set working dir
WORKDIR /var/frontend
# Command to run
CMD serve -s /var/frontend
# Tell docker the port
EXPOSE 5000
```
Truy cập thư mục `srv/frontend`

Chạy `docker build ./` để test build
Sau đó chạy `docker run -i -t -p 3000:5000 [built-ID]` chạy cái build id

Ok rồi thì có thể stop & remove cái container đó đi (kiểm tra container `docker ps -a`)

## Cài Nginx
Cài đặt Nginx để tạo load React chạy trên domain

Pull nginx từ docker hub `docker pull nginx:latest`

Tạo file nginx.conf

`nano nginx.conf`

```
user	nginx;
worker_processes	1;

error_log	/var/log/nginx/error.log warn;
pid	/var/run/nginx.pid;

events {
	worker_connections  1024;
}


http {
	include	/etc/nginx/mime.types;
	default_type	application/octet-stream;

	log_format	main	'$remote_addr - $remote_user [$time_local] "$request" '
										'$status $body_bytes_sent "$http_referer" '
										'"$http_user_agent" "$http_x_forwarded_for"';

	access_log	/var/log/nginx/access.log  main;

	sendfile	on;
	#tcp_nopush	on;

	keepalive_timeout	65;

	gzip	on;
  gzip_http_version 1.0;
  gzip_proxied      any;
  gzip_min_length   500;
  gzip_disable      "MSIE [1-6]\.";
  gzip_types        text/plain text/xml text/css
                    text/comma-separated-values
                    text/javascript
                    application/x-javascript
                    application/atom+xml;
	server {
		listen 80;
		index index.html;
		server_name DOMAIN_NAME;
		error_log  /var/log/nginx/error.log;
		access_log /var/log/nginx/access.log;
		root /srv/www/frontend;

		upstream nodeapp {
			server api: 5000
			server frontend: 5000
		}

		location / {
			proxy_pass http://nodeapp;
			proxy_redirect     off;
			proxy_http_version 1.1;
			proxy_set_header Upgrade $http_upgrade;
			proxy_set_header Connection 'upgrade';
			proxy_set_header Host $host;
			proxy_cache_bypass $http_upgrade;
		}
	}
}
```

Cấu hình nginx: sẽ chạy trên port web 80, file `index.html` lại thư mục /srv/www/frontend

Nano Dockerfile

```Dockerfile
# Set nginx base image
FROM nginx

COPY docker-entrypoint.sh /
ENTRYPOINT ["/docker-entrypoint.sh"]

# Copy custom configuration file from the current directory
COPY nginx.conf /etc/nginx/nginx.conf

VOLUME ["/etc/nginx/ssl", "/etc/nginx/psw"]

# Append "daemon off;" to the beginning of the configuration
# in order to avoid an exit of the container
RUN echo "daemon off;" >> /etc/nginx/nginx.conf

# Expose ports
EXPOSE 443

# Define default command
CMD service nginx start
```

Dockerfile sẽ gọi docker-entrypoint để apply nginx.conf

Trường hợp permission denied thì phải chạy `chmod +x docker-entrypoint.sh`


## Docker compose
Cài đặt version docker compose mới nhất (hiện tại là 1.17.1)
`sudo curl -o /usr/local/bin/docker-compose -L "https://github.com/docker/compose/releases/download/1.17.1/docker-compose-$(uname -s)-$(uname -m)"`

Cấu hình permission cho ubuntu.
`sudo chmod +x /usr/local/bin/docker-compose`

Kiểm tra phiên bản
`docker-compose -v`



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

Cuối cùng tạo file `docker-compose.yml` ở root/

## Docker-compose.yml

```yml
version: '3'
services:
  nginx:
    build: ./nginx
    links:
      - api:api
      - frontend:frontend
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /etc/nginx/psw:/etc/nginx/psw
      - /etc/nginx/ssl:/etc/nginx/ssl
    environment:
      - DOMAIN_NAME=my-domain-name.com
  api:
    build: .
    links:
      - mongodb
    ports:
      - "5000"
    volumes:
      - /srv/www/api:/srv/www/api
    environment:
      - MODE=prod
  frontend:
    build: /srv/www/frontend
    ports:
      - "3000"
    volumes:
      - /srv/www/frontend:/srv/www/frontend
    environment:
      - MODE=prod
  database:
    image: mongo:latest
    container_name: "database"
    restart: always
    environment:
      - MONGO_DATA_DIR=/database/db
      - MONGO_LOG_DIR=/dev/null
    volumes:
      - ./database/db:/database/db
    ports:
      - 27017:27017
    command: run -d --name database -p 27017:27017 -v ~/database:/data/db mongo --auth
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