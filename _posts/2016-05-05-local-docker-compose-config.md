---
layout: post
title:  "Local docker compose configuration"
date:   2016-05-05 3:30:21 -0700
categories: docker docker-compose blog rails
---
I've been exposed to Docker in the past and decided to get my Rails 4 app running in Docker with Postgres on the backend and nginx as a reverse proxy. I probably spent too much time trying to get the image size smaller than 876 MB (My app takes only 37 MB of that). So I'll take some time to document how I got the application (Postgres, Rails, Nginx) working with Docker with the files and images I used.

I start with a Dockerfile at the Rails root. You'll noticed my Dockerfiles don't have a CMD or ENTRYPOINT command. I'm currently using docker-compose to manage the relationship between the containers (Postgres, Rails, Nginx) and it handles the command with parameters as well.

```
FROM centurylink/alpine-rails
MAINTAINER Scott Stephenson <macsteps@gmail.com>

ENV APP_HOME /usr/src/lnat_app

RUN mkdir -p $APP_HOME
WORKDIR $APP_HOME

COPY Gemfile $APP_HOME
COPY Gemfile.lock $APP_HOME
RUN bundle install

COPY . $APP_HOME
```

Using alpine-rails, the image is 867 MB. Using ruby:2.2, the image was 893 MB. Not a huge difference, except perhaps for larger teams, where having almost 1 GB files flying back and forth exhausts network resources. I also tried putting rvm on debian:jessie (to avoid using Ruby 2.1, which probably wouldn't have matter in my case, but I persisted), which was still much larger than both ruby and alpine-rails.

I then created a ```nginx``` directory at Rails root. In this directory I put an nginx.conf file:

```
user                  www www;
worker_processes      1;  ## Default: 1
error_log             /var/log/nginx/error.log;
pid                   /var/log/nginx/nginx.pid;
worker_rlimit_nofile  8192;
daemon                off; # so nginx startup doesn't exit causing docker container to stop/exit

events {
  worker_connections  1024;  ## Nginx complains if not set here.
}

http {

  upstream the_app {
    server <docker-machine-ip>:3000;
  }

  server {
    listen 80;

    location / {
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_pass http://the_app;
    }
  }

}
```

Shutting off daemon mode is important, because Docker is looking to run programs in the foreground and most Linux services are designed to run in the background (daemon mode). The daemon mode can also be shutoff in the ```docker-compose.yml``` file instead. For example, if you want the nginx.conf file to be more portable (e.g. run it outside of Docker & in daemon mode), you might use this method instead. I didn't setup an access log at this point, just the error.log, which will actually be a symbolic link to ```STDERR``` (default of nginx image).

Next I need a ```Dockerfile``` for the nginx image, also in ./nginx (not to confuse with the app's Dockerfile at Rails root):

```
FROM nginx
MAINTAINER Scott Stephenson <macsteps@gmail.com>

RUN apt-get update -y && groupadd www && useradd -u 999 -g www www && \
    chown -R www:www /etc/nginx && chown -R www:www /var/log/nginx
COPY nginx.conf /etc/nginx
```

Then I put a docker-compose.yml file also at the Rails root:

```
version: '2'
services:
  db:
    image: postgres:9.4.1
    ports:
      - "5432:5432"
    volumes:
      - .:/etc/postgresql
      - .:/var/lib/postgresql

  app:
    build: .
    command: bin/rails server --port 3000 --binding 0.0.0.0
    ports:
      - "3000:3000"
    links:
      - db
    volumes:
      - .:/usr/src/lnat_app

  nginx:
    build: ./nginx
    command: service nginx start
    ports:
      - "80:80"
    links:
      - app
```

In this case, ```docker-compose``` handles the linking of the containers together (i.e. nginx -> app -> db). I run Postgres in development. Nginx and Rails are using their default ports, 80 and 3000, respectively. I'm able to use ```service nginx start```, because I turned off daemon mode in nginx.conf. Otherwise, you'll have to use ```nginx -g "daemon off"```.

With this in place, I just need to build, create/setup the database, and launch the containers.

Run from Rails root:

```
docker-compose build
```

You'll see Docker working step-by-step to build the images. When complete, create and setup the database:

```
docker-compose run -e RAILS_ENV=development app rake db:create db:setup
```

Rake outputs the results of setting up the database, running migrations, and seeding with sample data. Time to launch the containers.

If you want to view the logs immediately, use:

```
docker-compose up
```

If you want to continue working, then launch in daemon mode:

```
docker-compose up -d
```

Point your browser to:

```
http://<docker-machine-ip
```

If you're on a Mac and using the (Docker Toolbox)[https://docs.docker.com/mac/step_one/], just type:

```
docker-machine ip
```

If you get any errors regarding find the ip, toss this into your ~./bash_profile and reload it:

```
eval "$(docker-machine env default)"
source ~/.bash_profile
```

That's it for getting a development 3-tier application running with (Docker Compose)[https://docs.docker.com/compose/]. There's so much to do. Within the next couple weeks, I intend to try out (Docker Swarm)[https://docs.docker.com/swarm/] and perhaps use more ENVIRONMENT variables, with (docker-gen)[https://github.com/jwilder/docker-gen] to get around the messy <docker-machine-ip> in nginx.conf. Always much to do.
