Create and run a docker image
```
docker run -i -t ruby:2.4 bash
touch /tmp/test  ls /tmp
exit
```
`docker run -i -t ruby:2.4 bash -c 'touch /tmp/test; ls /tmp'`

The created file doesn't exist
```
docker run -i -t ruby:2.4 bash
ls /tmp  exit
```
`docker run -i -t ruby:2.4  bash -c 'ls /tmp'`

Map image volumes(-v) to local directory for persistence
Also cleans up(--rm) after run
`docker run -i -t --rm -v /tmp:/tmp ruby:2.4 bash -c 'touch /tmp/test; ls /tmp'`

Now the file persists and exists locally too
`docker run -i -t --rm -v /tmp:/tmp ruby:2.4  bash -c 'ls /tmp'`
`ls /tmp`

Create layers/build images
On one shell(SHELL-A) run
```
docker run -i -t --rm --name dev ruby:2.4 /bin/bash
apt-get update
apt-get -y upgrade
apt-get -y install libidn11-dev nodejs
apt-get -y install nano
```

On another shell(SHELL-B) run
`docker commit -p dev vyk:2.4 `

Exit SHELL-A
`exit`

run
`docker run -t -i --rm vyk:2.4 /bin/bash`

Create volume
`docker volume create gems`

Use volume
```
docker run -i -t -d \
-v gems:/usr/local/bundle \
-v ~/.ssh/config:/root/.ssh/config \
-v ~/.ssh/id_rsa_github:/root/.ssh/id_rsa  \
-v $(pwd):/usr/src/app \
-w /usr/src/app \
--name dev \
--rm \
vyk:2.4
```
Check  that the volume is created
`docker exec -i -t dev /bin/bash -c 'ls /usr/local/bundle'`

Run command on container(SHELL-A)
```
docker exec -i -t dev /bin/bash
bundle install rails
rails new
bundle install
bundle exec rails server -b 0.0.0.0
```

Check  that gems are on volume(SHELL-B)
`docker exec -i -t dev /bin/bash -c 'ls /usr/local/bundle'`

Check that you can connect to rails on the container(SHELL-B)
`docker exec -i -t dev /bin/bash -c 'curl http://localhost:3000'`

Check that you can't connect locally though
`curl http://localhost:3000`

Tail the logs
`docker exec -t -i dev tail -f log/development.log`

Use Dockerfile
In a file named Dockerfile, preferably in a blank directory to reduce context, write
```
FROM python:2-slim
RUN pip install --upgrade pip
RUN pip install --upgrade youtube-dl
ENV APP_HOME /user/src/app
RUN mkdir -p $APP_HOME
WORKDIR $APP_HOME
ENTRYPOINT ["youtube-dl"]
CMD ["--help"]
```
and save the file
Now build the container
`docker build -t yd .`

It's cached so second build is quicker
`docker build -t yd .`

Download a video using local temp as a volume
`docker run -i -t --rm -v /tmp/:/usr/local/app yd https://www.youtube.com/watch\?v\=rt2h8jSdxHY`

Locate the video locally
`ls /tmp/yd`

Ports and Volumes
create 2 Volumes
`docker volume create ws-mysql`
`docker volume create ws-bundle`

create a mysql container using the volumes
`docker run -d -v ws-mysql:/var/lib/mysql --restart=always -e MYSQL_ROOT_PASSWORD=password --name ws-mysql mysql:5.7`

use the mysql volume and gem volume to run rails
```
docker run -i -t -v ws-bundle:/usr/local/bundle -v /tmp/app:/app --name ws-app --link ws-mysql:mysql --rm vyk:2.4 /bin/bash

cd /app
gem install rails
rails new -d mysql .
rails server
```

map the port for local access
`docker run -i -t -v ws-bundle:/usr/local/bundle -v /tmp/app:/app -p 8888:3000 -w /app --name ws-app --link ws-mysql:mysql --rm vyk:2.4 /bin/bash -c 'rails server'`

access rails web remotely
`docker exec -i -t ws-app curl http://localhost:3000`

access rails web locally
`curl http://localhost:8888`

TODO: use database volume
`docker exec -i -t ws-app rake db:create`

currently getting error
```
docker run -i -t -v ws-bundle:/usr/local/bundle -v /tmp/app:/app -p 8888:3000 -w /app --name ws-app --link ws-mysql:mysql --rm vyk:2.4 /bin/bash
root@85776e98b04d:/app# rake db:create
#<Mysql2::Error: Can't connect to local MySQL server through socket '/var/run/mysqld/mysqld.sock' (2)>
Couldn't create database for {"adapter"=>"mysql2", "encoding"=>"utf8", "pool"=>5, "username"=>"root", "password"=>nil, "host"=>"localhost", "database"=>"app_development"}, {:charset=>"utf8"}
(If you set the charset manually, make sure you have a matching collation)
Created database 'app_development'
#<Mysql2::Error: Can't connect to local MySQL server through socket '/var/run/mysqld/mysqld.sock' (2)>
Couldn't create database for {"adapter"=>"mysql2", "encoding"=>"utf8", "pool"=>5, "username"=>"root", "password"=>nil, "host"=>"localhost", "database"=>"app_test"}, {:charset=>"utf8"}
(If you set the charset manually, make sure you have a matching collation)
Created database 'app_test'
```
