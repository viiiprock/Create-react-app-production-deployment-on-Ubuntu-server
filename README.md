# Create-react-app production deployment

My background is UX/UI designer/none-tech PM, so basically the stuffs about server is kinda technical difficulty. Sometimes I feel stress and struggling in my brain, but yes, it's excited.

Well, the context is, I have to deploy my app in my Ubuntu server: The frontend is built from create-react-app, the node API is run within PM2 watching on top, Nginx load balancer to proxy those app, and the Mongodb behind.

## Prepare to start.
- You need a server (off course)
- Install Ubuntu (currently lts 16.04)

## Containers structure
I think the `/srv` is a good to contain you app because it's a blank directory.

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
│
├─ api/
│ ├─ build
│ └─ Dockerfile
│
└─ docker-compose.yml
```

You could `ssh` to server as root admin to get rid of `sudo` command on terminal.

## Install Docker and docker-compose

Install docker

On terminal
```t
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
apt-get update
apt-get install -y docker-ce
```
[Learn more about Docker CE](https://docs.docker.com/engine/installation/linux/docker-ce/ubuntu/)

Install docker-compose

On terminal
```t
curl -o /usr/local/bin/docker-compose -L "https://github.com/docker/compose/releases/download/1.17.1/docker-compose-$(uname -s)-$(uname -m)"
chmod +x /usr/local/bin/docker-compose
```
You can check docker compose version with `docker-compose -v`

## Prepair your react app to serve dynamically
I use `create-react-app` starter kit for my front end, and to serve the built react application, I prefer to use `express` to run under `node`
Open your React app and add dependecies (I decide to add in `devDependencies`)

```json
{
  "devDependencies": {
    "body-parser": "^1.18.2",
    "express": "^4.16.2",
    "path": "^0.12.7"
  }
}
```

Create `serve.js` in the app directory.
<img src="1.png" style="width: 250px;">

**server.js**
```js
  const express = require('express');
  const bodyParser = require('body-parser')
  const path = require('path');
  const app = express();

  app.disable('x-powered-by');
  app.use(express.static(path.join(__dirname, 'build')));

  app.get('/', function (req, res) {
    res.sendFile(path.join(__dirname, 'build', 'index.html'));
  });

  app.listen(
    process.env.PORT || 5000,
    function () {
      console.log(`Server start on http://localhost:5000`)
    }
  );
```
Open terminal and run the command like `node server.js`
You will see you app's running on the `localhost:5000` absolutely.
Push your code to your repository.

## Git your repositories
The optional convenience way to get your code on server is to pull code from your repositories on Bitbucket, Github...You would prefer Docker hub repo, it's an option.
```t
apt-get update
apt-get install git
```
Config your Git

```t
git config --global user.name "Your Name"
git config --global user.email "youremail@domain.com"
```
To edit you Git config, use `nano ~/.gitconfig`

Clone your repo to `/srv`

```t
mkdir /srv/frontend
cd /srv/frontend
git clone
```

## Node and PM2
Create a directory name `node` in `/srv`
Create `Dockerfile`

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