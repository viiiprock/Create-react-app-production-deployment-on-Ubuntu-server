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
database/

nginx/
├─ Dockerfile
└─ nginx.conf

nodeapp/
├─ Dockerfile - chạy pm2
├─ process.json
│
├─ api/
│ ├── package.json
│ └── index.js
│
└─ frontend/
	├── package.json
	└── index.html

docker-compose.yml

```

## Cài đặt môi trường ban đầu
- Cài Ubuntu (lts 16)
- Cài Docker.
- Cài Ajenti (không bắt buộc) để quản lý file, có thể dùng cpanel, git...


## Docker compose
Cài đặt version docker compose mới nhất (hiện tại là 1.17.1)
`sudo curl -o /usr/local/bin/docker-compose -L "https://github.com/docker/compose/releases/download/1.17.1/docker-compose-$(uname -s)-$(uname -m)"`

Cấu hình permission
`sudo chmod +x /usr/local/bin/docker-compose`

Kiểm tra phiên bản
`docker-compose -v`


* Kiểm tra package cần cài trên [Docker Hub](https://hub.docker.com)

## Cài mongodb

```t
mkdir ~/database
sudo docker run -d -p 27017:27017 -v ~/database:/data/db mongo
```
có thể đặt Port khác `CUSTOM_PORT:27017` nếu cần

Kiểm tra danh sách docker containers `sudo docker ps`

Do tên container tự generate nên có thể đổi tên bằng cách `sudo docker renam old-name database`

Chạy mongodb auth `sudo docker run -d --name database -p 27017:27017 -v ~/database:/data/db mongo --auth`

Truy cập mongo `docker exec -it database mongo admin`

Tạo tài khoản root `db.createUser({ user: 'admin', pwd: 'password', roles: [ { role: "root", db:"admin" } ] });`

Thoát mongo bằng lệnh `exit`

Chạy mongo theo tài khoản đã thực hiện `docker exec -it database mongo -u "admin" -p "password" --authenticationDatabase "admin"`
Tạo tài khoản cho db

```js
use mydatabase
db.createUser ({
  user: "admin",
  pwd: "admin123456",
  roles: [ { role: "root", db: "admin" } ]
});

```

Dừng chạy database container `docker stop database`, remove container `docker rm database` (phải remove nếu không sẽ bị conflict).


## Cài Nginx

Pull nginx từ docker hub `docker pull nginx:latest`

```t
mkdir nginx
cd nginx
nano docker-entrypoint.sh
```
Copy dòng vào docker-entrypoint.sh

```sh
#!/bin/bash -e
sed -i s/DOMAIN_NAME/$DOMAIN_NAME/g /etc/nginx/nginx.conf
cat /etc/nginx/nginx.conf
exec "$@"
```

Lưu và exit

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

	server {
		listen 80;
		index index.html;
		server_name DOMAIN_NAME;
		error_log  /var/log/nginx/error.log;
		access_log /var/log/nginx/access.log;
		root /srv/www/frontend;

		upstream nodeapp {
			server api:8080
			server frontend: 3000
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
        - "8080"
      volumes:
        - /srv/
      environment:
        - MODE=prod
    frontend:

    mongodb:
      image: mongo:latest
      container_name: "mongodb"
      restart: always
      environment:
        - MONGO_DATA_DIR=/data/db
        - MONGO_LOG_DIR=/dev/null
      volumes:
        - ./data/db:/data/db
      ports:
        - 27017:27017
      command: mongod --auth
```
