# docker-training

## Prerequisites

Install Docker (Community Edition) on your system, if not present already <https://docs.docker.com/install/>. Check that `docker version` is returning a proper result for both the client and server.

Install Docker Compose on your system, if not present already <https://docs.docker.com/compose/install/>. Check that `docker-compose version` is returning a proper result.

Install the Dive tool for exploring Docker images and layers <https://github.com/wagoodman/dive>.

## Exersise 1 - Working with containers

### The basics commands to start with
Open a shell session on your system and see that`docker help` and `docker <cmd> help` are your friend. 
They display information about the possibilities of docker. 
Important commands are `docker images` to get a list with the current images, `docker ps` to get a list with de current active containers and `docker ps -a` to get a list with all the containers including the inactive. 
When you've never run a Docker container before the resulted list of previous commands will be empty. 
Let's change that now

### Starting a container
Use `docker run` to start a container from the nginx image, a webserver, and make it available at port 88.
Note that the first time you'll start a container Docker will download the needed Docker image from the Docker repository.
You need the flag `-d` to run the server as deamon (the command prompt becomes unresponsive) and the flag `-p hostport:guestport` for port mapping. 
Nginx within the container serves at port 80.
Use `—name` to give your container a manageable name instead of a random ID. 
Use a browser to access the nginx webserver `http://<dockerhost>:88/` and see the default web page being served.

### Looking inside a container
By default you have no insight in what is happening inside a container. 
But you are able to execute commands inside a running container. 
Now use `docker exec -ti <container-id> bash` to open a shell in the active container. 
Check the filesystem with `ls`, with `ps -ef` the active processes, with `hostname` the hostname and with `hostname -i` the assigned ip-address of the container. (end shell with `exit`)

### Stopping and restarting a container
Stop the docker container with `docker stop`. Make sure that the webpage is not available any more.
With `docker ps -a` you can see all the containers and their status.
Every time you execute `docker run` a new container gets created. 
After a run you can run the same container with the assigned name (`docker start name`) to prevent the creation of a huge amount of containers. 
With the flag `-d` the container runs as a deamon and you need to stop it explicitly. 
Restart the container now.

### Removing containers and images
Remove the previously started docker container with `docker rm`. 
Use the `-f` flag if the container is still running.
Remove the nginx image with `docker rmi`. 
This only works if the containers based on the image are removed. 
If this is not the case, use `docker ps -a` and `docker rm` to remove the remaining.

### Volume mounting
We are going to do something more exiting, we are going to serve our own (static) webpage `<project-directory>/web-app/src` from the nginx container. 
Start a new container with a port mapping and a mount to `/usr/share/nginx/html/` using `-v hostfolder:guestfolder:mountoptions`. 
In this case you can use `ro` for mountoptions, so it’s read-only. 
Use `/usr/share/nginx/html` as guestfolder (default webcontent directory of nginx) and `<project-directory>/web-app/src` as hostfolder. 
Note not to use relative paths.
Check the webpage with the browser.


## Exersise 2 - Creating images

In this part of the workshop we will be deploying a three tier application that comprises of a web server hosting an angular web application, a Java based REST service and a mysql database. In this part of the workshop we will create a Docker image for the frontend and backend layer. There is no dedicated Docker image for de database layer needed as the database tables of the application will be created by the hibernate layer of the REST service upon startup, and the database can be created by the docker image it self.

Step 1. Run MySQL

The Java REST service expects a mysql database with the name `cddb_quintor` and a user with the same name and `quintor_pw` as password.

`MYSQL_ROOT_PASSWORD=quintor_pw
MYSQL_DATABASE=cddb_quintor
MYSQL_USER=cddb_quintor
MYSQL_PASSWORD=quintor_pw`

Use the standard version 5.6 mysql image from Docker hub (https://hub.docker.com/_/mysql/) for this. There are environment variables available to let the mysql docker container create the database and an user. Give the mysql container the name `cddb_mysql` so it can be linked later on. Bind the mysql port (3306) to a local port and check that the database is available.

Step 2. Create an docker image of the Java REST service

The Java REST service is a simple JavaEE web app that is ready available in the three-tier-app/backend folder. This WAR file can be run with tomcat 8.5, so create a Docker image based on https://hub.docker.com/_/tomcat/ using a Docker file. Copy the war file to `/usr/local/tomcat/webapps/cddb.war` in the image.

Link the `cddb_mysql` container to mysql so the application can connect to mysql by using the link option: `--link cddb_mysql:mysql`. Give the backend container the name `cddb_backend` so it can be linked later on. Bind the tomcat port (8080) to a local port and check that the REST service is available using a browser or other tool `http://<dockerhost>:<bindport>/cddb/rest/`.

Step 3. Create an Docker image of the Angular web app

The web application is a simple Angular application compising only of static files to be served. This can be done easily using nginx and copy the webapp content.
`COPY ./ /usr/share/nginx/html`
To make the REST endpoints easily accessible form the browser an nginx configuration is provided (`nginx.conf`) to create a reverse proxy that proxies the REST backend service.
`COPY nginx.conf /etc/nginx/`
Therefor also link the `cddb_backend` container to `cddb_backend` so the nginx reverse proxy can connect to the Java backendby using the link option: `--link cddb_backend:cddb_backend`. Bind the nginx port (80) to a local port and check that the web application is available and working using a browser `http://<dockerhost>:<bindport>/`.

Step 4.

Use volume mounting to store the mysql data locally. After removing the mysql container the stored data should be available again when a container is create with the same volume mount.

## Exersise 3 - Use Docker Compose to run the three tier application at once

Docker compose give you the possibility to run multiple containers that are depended on each other at once. Command line options that are needed to link containers, mount volumes and expose ports can be defined is one configuation file.

Create a docker compose yaml configuration file to start up the three tier application at once. Documentation can be found here: https://docs.docker.com/compose/overview/
