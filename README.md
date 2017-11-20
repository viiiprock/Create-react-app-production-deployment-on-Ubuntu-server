# Create-react-app production deployment

This is the first time I working on server stuffs, so my context is that I have to deploy my app in my Ubuntu server: The frontend is built from create-react-app, the node API is run with PM2 process manager on top, Nginx load balancer to proxy those app, and the Mongodb behind.

This article is noted when I proccessed my work, save for later for me as well. You could see this article as an example.

## Prepare to start.
- You need a server (off course)
- Install Ubuntu (currently lts 16.04)

## Containers structure
I think the `/srv` is a good place to contain you app because it's a blank directory.

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

[Install docker](https://docs.docker.com/engine/installation/linux/docker-ce/ubuntu/)

[Learn more about Docker CE](https://docs.docker.com/engine/installation/linux/docker-ce/ubuntu/)

Install docker-compose, open terminal
```t
curl -o /usr/local/bin/docker-compose -L "https://github.com/docker/compose/releases/download/1.17.1/docker-compose-$(uname -s)-$(uname -m)"
chmod +x /usr/local/bin/docker-compose
```
You can check docker compose version with `docker-compose -v`

## Node and dependencies
I need to install node and dependencies
Create `Dockerfile` in `/srv/node`

```Dockerfile
# From Ubuntu image
FROM ubuntu:16.04
# Clean and update
RUN apt-get clean && apt-get update
# Install dependencies
RUN apt-get -y install curl && \
    apt-get -y install wget && \
    apt-get -y install nginx && \
    apt-get -y install apt-utils && \
    apt-get autoremove -y
# Install node v.8
RUN curl -sL https://deb.nodesource.com/setup_8.x | bash - && \
    apt-get install --yes nodejs
```

## Prepair your react app to serve dynamically
I use `create-react-app` starter kit for my front end, and to serve the built react application, I prefer to use `express` to run under `node`.
Open your React app and add dependencies.

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

<img src="1.png">

**server.js**
```js
  const express = require('express');
  const bodyParser = require('body-parser')
  const path = require('path');
  const app = express();

  app.disable('x-powered-by');
  app.use(express.static(path.join(__dirname, 'build')));

  app.get('*', function (req, res) { //need to declare a "catch all" route on your express server that captures all page requests and directs them to the client
    res.sendFile(path.join(__dirname, 'build', 'index.html'));
  });

  app.listen(
    process.env.PORT || 5000,
    function () {
      console.log(`Frontend start on http://localhost:5000`)
    }
  );
```
Add proxy to the `package.json`

```json
{
  "proxy": "http://localhost:5000",
}
```

**TL;DR** You can add proxy as `0.0.0.0:5000` if you get proxy notification in your console i you have some custom files in `public` directory need to proxy.

Open terminal and run the command like `node server.js`

You will see you app's running on the `localhost:5000` absolutely.

Commit and push your code.

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
cd /srv
git clone [myrepo@link]
```
Add

So, I want those pull, build and run automatically with Docker, I need to create `Dockerfile` on the app folder.

And frontend Dockerfile is like:

```Dockerfile
# Based image from node
FROM node:8
# The base node image sets a very verbose log level.
ENV NPM_CONFIG_LOGLEVEL warn
# Set working dir
WORKDIR /srv/frontend
# Bundle app source
COPY . /srv/frontend
# Install app dependencies
RUN npm install
# Build app
RUN npm run build
# Command to run
CMD node server.js
# Tell docker the port
EXPOSE 5000
```
## Nginx
Nginx works as load balancer on the server.
In `/srv/nginx` directory, create `Dockerfile`

```Dockerfile
# Set nginx base image
FROM nginx
# Remove the default Nginx configuration file
RUN rm -v /etc/nginx/nginx.conf
# Copy custom configuration file from the current directory
COPY nginx.conf /etc/nginx/nginx.conf
# Append "daemon off;" to the beginning of the configuration
# in order to avoid an exit of the container
RUN echo "daemon off;" >> /etc/nginx/nginx.conf
# Expose ports
EXPOSE 80
# Define default command
CMD service nginx start
```
Then I create `nginx.conf` file

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

	upstream node-app {
		least_conn;
		server React:5000 weight=10 max_fails=3 fail_timeout=30s;
	}

  #####################
  # REACT
  #####################
	server {
		listen 80;
		root /srv/frontend;
		index index.html index.htm;
		server_name my-domain.com;
		# Routes without file extension
		location / {
			proxy_pass http://node-app/;
			proxy_http_version 1.1;
			proxy_set_header Upgrade $http_upgrade;
			proxy_set_header Connection 'upgrade';
			proxy_set_header Host $host;
			proxy_cache_bypass $http_upgrade;
		}
	}
}
```

## docker-compose.yml
I use docker compose to run those docker commands once.
So create `docker-compose.yml`

```yaml
version: '3'
services:
  ####################
  # NODE
  ####################
  node:
    container_name: Node
    build         : ./node

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
    ports:
      - "80:80"
      - "443:443"

  #####################
  # REACT
  #####################
  react:
    image         : react
    container_name: React
    # Dockerfile location
    build         : ./frontend
    ports         :
      - "5000"
```

Time to build and run

`cd /srv` in your terminal

Update, build, run your app for the first time

```t
docker-compose up --build -d
docker-compose up
```

Yeah, now, you could go to `my-domain.com` and see how your app run. Yummy, right?!

If you have no thing to update, run command like this

```t
docker-compose build
docker-compose up
```

*Tips* Some essencial docker commands
- View containers in directory `docker ps`
- View all containers `docker ps -a`
- Stop container `docker stop [container-name]`
- Kill container `docker rm [container-name]`
- Stop all containers `docker stop $(docker ps -a -q)`
- Kill all containers `docker rm $(docker ps -a -q)`
- Kill all images `docker rmi $(docker images -q)`

[more](https://www.digitalocean.com/community/tutorials/how-to-remove-docker-images-containers-and-volumes)


## Use PM2
PM2 is a cool process management for Node application within many utility benefits, first of all I need PM2 to watch my node apps run and restart in case crashed.
For now, to use PM2 is just simply to install PM2 and change `node server.js` to `pm2 start server.js` in `/srv/frontend/Dockerfile`.
```Dockerfile
# Based image from node
FROM node:8
RUN npm install -g pm2
# The base node image sets a very verbose log level.
ENV NPM_CONFIG_LOGLEVEL warn
# Set working dir
WORKDIR /srv/frontend
# Bundle app source
COPY . /srv/frontend
# Install app dependencies
# To mitigate issues with npm saturating the network interface we limit the number of concurrent connections
RUN npm config set maxsockets 5 && \
    npm config set progress false && \
    npm install && \
    npm run build

# Command to run
CMD pm2 start --no-daemon server.js
# Tell docker the port
EXPOSE 5000
```

## Mongodb
Time to prepare Mongodb for the API

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

## Node API


## Next step

This is the next step I would do
1. Improve/refactor Docker enviroment, reference: http://nodesource.com/blog/8-protips-to-start-killing-it-when-dockerizing-node-js/
2. Setting https for your web application
3. Use IPv6
4. Boost up with http/2