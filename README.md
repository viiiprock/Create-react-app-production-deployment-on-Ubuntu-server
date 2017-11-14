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
node/
├── Dockerfile - chạy pm2
├── process.json
│
├── api/
│   └── src/
│       ├── package.json
│       └── index.js
│
└── frontend/
	 └── src/
				├── package.json
				└── index.html


database/

nginx/
├── Dockerfile
└── nginx.conf


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
mkdir ~/project_db
sudo docker run -d -p 27017:27017 -v ~/project_db:/data/db mongo
```
có thể đặt Port khác `CUSTOM_PORT:27017` nếu cần

Kiểm tra danh sách docker containers `sudo docker ps`

Do tên container tự generate nên có thể đổi tên bằng cách `sudo docker renam old-name project_db`

Chạy mongodb auth `sudo docker run -d --name project_db -p 27017:27017 -v ~/project_db:/data/db mongo --auth`

Chạy mongo `docker exec -it project_db mongo admin`

Tạo tài khoản root `db.createUser({ user: 'admin', pwd: 'password', roles: [ { role: "root", db:"admin" } ] });`

Thoát mongo bằng lệnh `exit`

Chạy mongo theo tài khoản đã thực hiện `docker exec -it project_db mongo -u "admin" -p "password" --authenticationDatabase "admin"`
Tạo tài khoản cho db

```js
use mydatabase
db.createUser ({
		user: "admin",
		pwd: "admin123456",
		roles: [ { role: "root", db: "admin" } ]
});

```

## Cài Nginx

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

## Docker-compose.yml

```
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
				- "3000"
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