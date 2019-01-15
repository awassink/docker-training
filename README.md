# docker-training

## Prerequisites

Install Docker (Community Edition) on your system, if not present already <https://docs.docker.com/install/>. Check that `docker version` is returning a proper result for both the client and server.

Install Docker Compose on your system, if not present already <https://docs.docker.com/compose/install/>. Check that `docker-compose version` is returning a proper result.

Install the Dive tool for exploring Docker images and layers <https://github.com/wagoodman/dive>.

## Exersise 1 - Working with containers

### The basics commands to start with
Open a shell session on your system and see that`docker help` and `docker help <cmd>` are your friend. 
They display information about the possibilities of docker. 
Important commands are `docker images` to get a list with the current images, `docker ps` to get a list with de current active containers and `docker ps -a` to get a list with all the containers including the inactive. 
When you've never run a Docker container before the resulted list of previous commands will be empty. 
Let's change that now

### Starting a container
Use `docker run` to start a container from the `nginx:alpine` image, a webserver, and make it available at port 88.
Note that the first time you'll start a container Docker will download the needed Docker image from the Docker repository.
You need the flag `-d` to run the server as deamon (the command prompt becomes unresponsive) and the flag `-p hostport:guestport` for port mapping. 
Nginx within the container serves at port 80.
Use `—-name` to give your container a manageable name instead of a random ID. 
Use a browser to access the nginx webserver `http://localhost:88/` and see the default web page being served.

### Looking inside a container
By default you have no insight in what is happening inside a container. 
But you are able to execute commands inside a running container. 
Now use `docker exec -ti <container-id> sh` to open a shell in the active container. 
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
If you change the text in the index.html file this change will directly be visible when you refresh the page in the browser.

### Explore the Dokcer image and its layers
With `dive` we can have a look into Docker images and how they are build up.
Now start exploring the NGINX image, `dive nginx:alpine`, and browse through the image layers and have a look at the changes made in each layer to the filesystem.


## Exersise 2 - Creating images and linking containers
In this part of the workshop we will be deploying a three tier application that comprises of a web server hosting an angular web application, a Java based REST service and a mysql database. 
For that we will create a Docker image for the frontend and backend layer. 
There is no dedicated Docker image for de database layer needed as the database tables of the application will be created by the hibernate layer of the REST service upon startup, and the database can be created by the docker image it self.
Lets get started!

### Run MySQL
The Java REST service expects a mysql database with the name `cddb_quintor` and a user with the same name and `quintor_pw` as password.

Use the standard version 5.6 mysql image from Docker hub <https://hub.docker.com/_/mysql/> for this. 
There are environment variables available to let the mysql docker container create the database and an user. 
Use the option `-e` of the `docker run` command to specify these. 
- `MYSQL_ROOT_PASSWORD=quintor_pw`
- `MYSQL_DATABASE=cddb_quintor`
- `MYSQL_USER=cddb_quintor`
- `MYSQL_PASSWORD=quintor_pw`

Give the mysql container the name of `cddb_mysql` so it can be linked later on. 
Bind the mysql port `3306` to a local port (e.g. `23306`) and check that the database is available using a SQL client.

### Create a docker image of the Java REST service
The Java REST service is a simple JavaEE web app that is available in the `three-tier-app/backend` folder. 
The WAR-file in the `target` folder can be run with tomcat 8.5, so create a Docker image based on <https://hub.docker.com/_/tomcat/> using a Dockerfile. 
In de Dockerfile specify to copy the war file to `/usr/local/tomcat/webapps/cddb.war` in the image.
Now build the Docker image using `docker build` and tag the created image using `docker tag`.

### Run the Java REST service
By default the backend container won't know where to connect to the mysql container, but we are able to specify that with linking. 
Link the `cddb_mysql` container to `mysql` so the application can connect to mysql, using the link option: `--link cddb_mysql:mysql`. 
Give the backend container the name `cddb_backend` so it can be linked later on. 
Bind the tomcat port `8080` to a local port (e.g. `28080`). 
Check with `docker logs` that the service started without errors and check that the REST service is available using a browser or other tool <http://localhost:28080/cddb/rest/>.
You should also see that the database tables of the application are created.

### Create a Docker image of the Angular web app
The web application is a simple Angular application comprising only of static files to be served. 
This can be done easily using a nginx image with copied webapp content, so create a Docker image based on <https://hub.docker.com/_/nginx/> using a Dockerfile.
In de Dockerfile specify to copy the `frontend/src` to `/usr/share/nginx/html` in the image.
To make the REST endpoints easily accessible from the browser an nginx configuration is provided `frontend/resources/nginx.conf` to create a reverse proxy that proxies the REST calls to the backend service.
In de Dockerfile specify to copy the `nginx.conf` to `/usr/share/nginx/html` in the image.
Now build the Docker image using `docker build` and tag the created image using `docker tag`.

### Run the Angular web app
Link the `cddb_backend` container to `cddb_backend` so the nginx reverse proxy can connect to the Java backend by using the `--link` option. 
Bind the nginx http port `80` to a local port (e.g. `20080`) and check that the web application is available and working using a browser <http://localhost:20080/>.

### Store the database files on the host system
By default the data written by processes in a container wil end up in the writable layer of the container.
This is fine as long as the container exists and the data is transient, but in case of our mysql database we want to keep our data longer than the lifetime of the container!
We can achieve this by mounting a host folder location in the container using the `-v` option.
Stop and remove the existing mysql database container.
Run the mysql container now with volume mounting to store the mysql data locally (guestfolder: `/var/lib/mysql`). 
Look at the folder on the host system and see that mysql is writing it's database files there.
Note: to recreate the database remove the backend container and run it again.
After entering some data in the application: remove the mysql container. 
The stored data should be available again when the container is created again with the same volume mount.

## Exersise 3 - Use Docker Compose to run the three tier application at once
In exersise 2 we were able to run the three tier application using three quite large `docker run` commands.
Wouldn't it be nice to do that more easily?
Docker Compose give you the possibility to run multiple containers that are depended on each other with all there options at once. 
Command line options that are needed to link containers, mount volumes and expose ports can be defined is one configuation file: `docker-compose.yaml`.

Create a `docker-compose.yaml` configuration file to start up the three tier application at once. 
Documentation can be found at <https://docs.docker.com/compose/overview/>
Note that the first time the mysql container is started it take more time, because the database files need to be created.
This time is longer that the backend takes to startup, which fails because mysql is not yet available.
Restarting the compose setup should fix this issue.
