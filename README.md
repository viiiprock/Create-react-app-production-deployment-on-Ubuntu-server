# Create-react-app production deployment on Ubuntu server

**!Warning** Document is in progress

Actually, my background is UX/UI designer/none-tech PM, so basically all the stuffs about server, it's kinda technical difficulty for me. Sometimes I feel stress and struggling in my brain, but yes, it's excited.

Well, the context is, I have to deploy my app in my Ubuntu server: The frontend is built from create-react-app, the node API is run within PM2 watching on top, Nginx load balancer to proxy those app, and the Mongodb (or Redis) behind.

## Prepare to start.
- You need a server (off course)
- Install Ubuntu (suggest currently lts 16.04)

## Containers structure
I think the `/srv` is a good location because it's a blank folder.

```
srv/
|
├─ node/
│ └─ Dockerfile
|
├─ frontend/
│ ├─ build
│ └─ Dockerfile
│
├─ mongo/
│ └─ Dockerfile
├─ api/
│ ├─ build
│ └─ Dockerfile
│
└─ docker-compose.yml
```

## Install Docker and docker-compose

Install docker

```t
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get update
sudo apt-get install -y docker-ce
```
[Learn more about Docker CE](https://docs.docker.com/engine/installation/linux/docker-ce/ubuntu/)

Install docker-compose

```t
sudo curl -o /usr/local/bin/docker-compose -L "https://github.com/docker/compose/releases/download/1.17.1/docker-compose-$(uname -s)-$(uname -m)"
sudo chmod +x /usr/local/bin/docker-compose
```
You can check docker compose version with `docker-compose -v`

##

## Node and PM2

## React App

## Node API

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

Hoàn tất, chạy lệnh `docker-compose up --build -d services`
stop, kill all containers
kill images
```
docker stop $(docker ps -a -q)
docker rm $(docker ps -a -q)
docker rmi $(docker images -q)
```
[more](https://www.digitalocean.com/community/tutorials/how-to-remove-docker-images-containers-and-volumes)