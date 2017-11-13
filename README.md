# Ubuntu_sever_setup
Docker: Mongodb, Nginx, Node flow

Đây là bài viết để lưu trữ, khi cần lôi ra coi cho nhanh.
*Ghi chú*: để thuận tiện thì nên `ssh` server trên máy local.

- Cài Ubuntu (lts 16)
- Cài Docker.

Kiểm tra package cần cài trên [Docker Hub](https://hub.docker.com)

## Docker compose
Cài đặt version docker compose mới nhất (hiện tại là 1.17.1)
`sudo curl -o /usr/local/bin/docker-compose -L "https://github.com/docker/compose/releases/download/1.17.1/docker-compose-$(uname -s)-$(uname -m)"`

Cấu hình permission
`sudo chmod +x /usr/local/bin/docker-compose`

Kiểm tra phiên bản
`docker-compose -v`

## Cài mongodb

```
mkdir ~/project_db
sudo docker run -d -p 27017:27017 -v ~/project_db:/data/db mongo
```
có thể đặt Port khác `CUSTOM_PORT:27017` nếu cần

Kiểm tra danh sách docker containers `sudo docker ps`

Do tên container tự generate nên có thể đổi tên bằng cách `sudo docker renam gen-name project_db`

Chạy thử `docker run mongo` ok!

Chạy mongodb auth `docker run --name project_db -d mongo --auth`

Nó sẽ báo lỗi `docker: Error response from daemon: Conflict. The container name "/project_db" is already in use by container "e1928890ed9a3893516e3e33dc67ff7e1ddd2456217c59399a6a3373aaf2a22b". You have to remove (or rename) that container to be able to reuse that name.`

Vì vậy cần xóa nó đi và tạo lại với chế độ auth

```
docker stop project_db
docker rm project_db

sudo docker run -d --name project_db -p 27017:27017 -v ~/project_db:/data/db mongo --auth
```
Chạy mongo `docker exec -it project_db mongo admin`

Tạo tài khoản root `db.createUser({ user: 'admin', pwd: 'password', roles: [ { role: "root", db:"admin" } ] });`

Thoát mongo bằng lệnh `exit`

Chạy mongo theo tài khoản đã thực hiện `docker exec -it project_db mongo -u "admin" -p "password" --authenticationDatabase "admin"`
Tạo tài khoản cho db

```
use mydatabase
db.createUser ({
    user: "admin",
    pwd: "admin123456",
    roles: [ { role: "root", db: "admin" } ]
});

```