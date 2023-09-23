# Inception

This project is an introduction to the DevOps world. Its purpose is to introduce us to the use of docker and docker-compose to deploy a small web server that use the NGINX server with a Wordpress website and a MariaDB database. It's a simple project, but it's very important to understand the DevOps basic concepts. 

## Grade: 100/100

# The Inception Guide
This project will cover concepts that we didn't see previously, so I recommend you to start backward to front in your PC, then you try it in the VM. On this page I'll leave a guide that will follow this order. At the end, you will have a VM with docker and docker-compose installed, and you'll be able to deploy a wordpress website with a mariadb database hosted with NGINX server.

## Table of Contents
- [1. The Containers](#1-the-containers)
	- [1.1. mariaDB](#11-mariadb)
		- [1.1.1. Dockerfile](#111-dockerfile)
		- [1.1.2. 50-server.cnf](#112-50-servercnf)
		- [1.1.3. setup.sh](#113-setupsh)
		- [1.1.4. Test the mariadb container](#114-test-the-mariadb-container)
	- [1.2. Wordpress](#12-wordpress)
		- [1.2.1. Dockerfile](#121-dockerfile)
		- [1.2.2. www.conf](#122-wwwconf)
		- [1.2.3. wp-config.php](#123-wp-configphp)
		- [1.2.4. setup.sh](#124-setupsh)
		- [1.2.5. Test the wordpress container](#125-test-the-wordpress-container)
	- [1.3. NGINX](#13-nginx)
		- [1.3.1. The Dockerfile](#131-the-dockerfile)
		- [1.3.2. server.conf](#132-serverconf)
		- [1.3.3. nginx.conf](#133-nginxconf)
		- [1.3.4. Test the nginx image](#134-test-the-nginx-image)
- [2. Docker-compose](#2-docker-compose)
	- [2.1. docker-compose.yml](#21-docker-composeyml)
	- [2.2. .env](#22-env)
	- [2.3. docker-compose test](#23-docker-compose-test)
- [3. The Makefile](#3-the-makefile)
- [4. The VM](#4-the-vm)
	- [4.1. VM creation](#41-vm-creation)
	- [4.2. Debian installation](#42-debian-installation)
	- [4.3. VM Setup](#43-vm-setup)
		- [4.3.1. Add user as sudo](#431-add-user-as-sudo)
		- [4.3.2. Enable Shared Folders](#432-enable-shared-folders)
		- [4.3.3. Install Docker and docker-compose](#433-install-docker-and-docker-compose)
		- [4.3.4. Install make and hostsed](#434-install-make-and-hostsed)
- [5. The Website](#5-the-website)
	- [5.1. Add your files in the VM](#51-add-your-files-in-the-vm)
	- [5.2. Start the containers](#52-start-the-containers)
	- [5.3. Credentials check](#53-credentials-check)
	- [5.4. mariaDB check](#54-mariadb-check)
	

## 1. The Containers

First of all, you need to understand the basic of `docker`. I'll leave a guide that helped me with the docker and start my first `container`.

Follow the link: https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04

**PS:** I very much recommend you follow the part of the guide above that teach how to create a non-root user for the docker. This will help you a lot in the future.

Now, we'll create our owns containers one per service. We'll create a folder with the service name, then inside we'll create the `Dockerfile` and the folder `conf` and `tools` when it's necessary.

I'll cover the important things that you need to know on each service, but inside the repository you can find the files with comments to see how it works too. Every time something new appears, I'll try to explain what it's, but the next time I'll only show the command. Therefore, the first services will have more explanations than the last one.

**Disclaimer:** The subject informs that we need to use the penultimate stable version of debian or alpine. The latest stable debian version in September 2023 is debian 12 (bookworm). You can check this [here](https://www.debian.org/releases/). Because all of that, I used the debian 11 (bullseye) for all images.

### 1.1. mariaDB
Files: [Dockerfile](https://github.com/waltergcc/42-inception/blob/main/inception/srcs/requirements/mariadb/Dockerfile), [conf/50-server.cnf](https://github.com/waltergcc/42-inception/blob/main/inception/srcs/requirements/mariadb/conf/50-server.cnf), [tools/setup.sh](https://github.com/waltergcc/42-inception/blob/main/inception/srcs/requirements/mariadb/tools/setup.sh)

#### 1.1.1. Dockerfile
1. Use the debian 11 (bullseye) image
```Dockefile
FROM debian:bullseye
```
2. Indicates that this container will be listening on port 3306
```Dockerfile
EXPOSE 3306
```
3. Update the system and install the mariadb-server only. The `--no-install-recommends` and `--no-install-suggests` flags are used to avoid installing unnecessary packages. I used the the commands with `&&` to avoid creating unnecessary layers. You can check more about the best practices [here](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/). the `rm -rf /var/lib/apt/lists/*` is used to clean the apt cache and avoid unnecessary files in the image.
```Dockerfile
RUN	apt update && \
	apt install -y --no-install-recommends --no-install-suggests \
	mariadb-server && \
	rm -rf /var/lib/apt/lists/*
```
4. Copy the configuration file to the container. The first argument is the file path in the host, and the second is the file path that will receive the file.
```Dockerfile
COPY	conf/50-server.cnf /etc/mysql/mariadb.conf.d/
```
5. Copy the setup script to the container and change its permissions
```Dockerfile
COPY	tools/setup.sh /bin/
RUN	chmod +x /bin/setup.sh
```
6. Execute the setup script then start the database server. At the end of the script, it'll call the `mysqld_safe` command to start the database server.
```Dockerfile
CMD	["setup.sh", "mysqld_safe"]
```
#### 1.1.2. 50-server.cnf
It's a default file without the commented lines. The important thing here is change this: `port=3306`.

#### 1.1.3. setup.sh
1. Start the database server
```bash
service mariadb start
```
2. To check if all is ok, we'll declare these variables in the own script, but in the final version we'll have a .env file with all the variables. So you need to remove these declaration lines.
```bash
DB_NAME=thedatabase
DB_USER=theuser
DB_PASSWORD=abc
DB_PASS_ROOT=123
```
3. Create the database and the users with its passwords and permissions.
```bash
mariadb -v -u root << EOF
CREATE DATABASE IF NOT EXISTS $DB_NAME;
CREATE USER IF NOT EXISTS '$DB_USER'@'%' IDENTIFIED BY '$DB_PASSWORD';
GRANT ALL PRIVILEGES ON $DB_NAME.* TO '$DB_USER'@'%' IDENTIFIED BY '$DB_PASSWORD';
GRANT ALL PRIVILEGES ON $DB_NAME.* TO 'root'@'%' IDENTIFIED BY '$DB_PASS_ROOT';
SET PASSWORD FOR 'root'@'localhost' = PASSWORD('$DB_PASS_ROOT');
EOF
```
4. Prepare to restart the server to apply the changes. The `sleep` command is used to avoid errors before stopping the server.
```bash
sleep 5
service mariadb stop
```
5. Restart the server with the command passed as argument in the `Dockerfile`.
```bash
exec $@
```
#### 1.1.4. Test the mariadb container
Now, you can build your container and tests it. Inside the folder `mariadb`, run the following command. `build` is the command to build the image, and `-t` is the tag name and `mariadb` is the name that I recommend and `.` indicates that the `Dockerfile` is in the current folder.
```bash
docker build -t mariadb .
```
Then, run the container with the following command. `run` is the command to run the container, `-d` is the flag to run the container in background, and `mariadb` is the name of the image that we want to run.
```bash
docker run -d mariadb
```
Now, run the following command to check if the container is running and get its ID.
```bash
docker ps -a
```
With the ID copied, run the next command to get inside the container. `exec` is the command to execute a command inside the container, `-it` is the flag to run the command in interactive mode, and `ID` is the ID of the container and `/bin/bash` is the command that we want to execute, in this case we want to use its terminal.
```bash
docker exec -it copiedID /bin/bash
```
Now, you are inside the container. Run the following command to check if the database is created correctly and running. 
```bash
mysql -u theuser -p thedatabase
```
if you see the the prompt `MariaDB [thedatabase]>` it means that all is ok. Too see the tables, run the following command. For now, we don't have any table, so it'll return an empty set, But at the end of the project, it'll have some tables created by wordpress.
```bash
SHOW TABLES;
```
Now, to exit mysql, run `exit` then run `exit` again to exit the container. So it's all working, then we'll clean our container test. To stop the container, remove it and the image run the following commands.
```bash
docker rm -f $(docker ps -aq) &&  docker rmi -f $(docker images -aq)
```

### 1.2. Wordpress
Files: [Dockerfile](https://github.com/waltergcc/42-inception/blob/main/inception/srcs/requirements/wordpress/Dockerfile), [conf/wp-config.php](https://github.com/waltergcc/42-inception/blob/main/inception/srcs/requirements/wordpress/conf/wp-config.php), [conf/www.conf](https://github.com/waltergcc/42-inception/blob/main/inception/srcs/requirements/wordpress/conf/www.conf), [tools/setup.sh](https://github.com/waltergcc/42-inception/blob/main/inception/srcs/requirements/wordpress/tools/setup.sh)

#### 1.2.1. Dockerfile
1. Use the debian 11 (bullseye) image
2. Indicates that this container will be listening on port 9000
3. Set a variable to use in the next commands. `ARG` is only avaliable in the build time.
```Dockerfile
ARG	PHPPATH=/etc/php/7.4/fpm
```
4. Update the system and install `ca-certificates`, `php7.4-fpm`, `php7.4-mysql`, `wget` and `tar`.
5. After the php installation, it's running, so we need to stop it to change the configuration file.
```Dockerfile
RUN	service php7.4-fpm stop
```
6. Copy the configuration file to the php folder, then change some values in the php config files. 
```Dockerfile
COPY	conf/www.conf ${PHPPATH}/pool.d/
RUN		sed -i 's/;cgi.fix_pathinfo=1/cgi.fix_pathinfo=0/g' ${PHPPATH}/php.ini && \
		sed -i "s/listen = \/run\/php\/php$PHP_VERSION_ENV-fpm.sock/listen = 9000/g" ${PHPPATH}/pool.d/www.conf && \
		sed -i 's/;listen.mode = 0660/listen.mode = 0660/g' ${PHPPATH}/pool.d/www.conf && \
		sed -i 's/;daemonize = yes/daemonize = no/g' ${PHPPATH}/pool.d/www.conf
```
7. Download the wordpress CLI, change its permissions and move it to the `bin/wp` folder.
```Dockerfile
RUN	wget --no-check-certificate https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar && \
	chmod +x wp-cli.phar && \
	mv wp-cli.phar /usr/local/bin/wp
```
8. Create some folders needed by the wordpress files and change its owner to `www-data` user.
```Dockerfile
RUN	mkdir -p /run/php/ && \
	mkdir -p /var/run/php/ && \
	mkdir -p /var/www/inception/

RUN	chown -R www-data:www-data /var/www/inception/
```
9. Copy the wp-config.php and the setup script to the container and change its permissions
10. Execute the setup script then start the php server. `--nodaemonize` is used to avoid the php server to run in background.
```Dockerfile
CMD	["setup.sh", "php-fpm7.4", "--nodaemonize"]
```
#### 1.2.2. www.conf
It's a default file that the most important thing here is change the user, group and port.
```conf
user = www-data
group = www-data
listen = 9000
```
#### 1.2.3. wp-config.php
It's a default file. The important thing here is change some lines to use our database. For now, we'll use test values, but after we need to change it to the .env variables.
```php
define( 'DB_NAME', getenv('thedatabase') );
define( 'DB_USER', getenv('theuser') );
define( 'DB_PASSWORD', getenv('abc') );
define( 'DB_HOST', getenv('mariadb') );
define( 'WP_HOME', getenv('https://login.42.fr') );
define( 'WP_SITEURL', getenv('https://login.42.fr') );

```
#### 1.2.4. setup.sh
1. Change the owner of the wordpress files to `www-data` user.
```bash
chown -R www-data:www-data /var/www/inception/
```
2. Move the wp-config.php file to the wordpress folder if it isn't there.
```bash
if [ ! -f /var/www/inception/wp-config.php ]; then
	mv /tmp/wp-config.php /var/www/inception/
fi
```
3. Create the temp var to be used now, but at the end, we'll have a .env file with all the variables. So you need to remove these declaration lines.
```bash
WP_URL=login.42.fr
WP_TITLE=Inception
WP_ADMIN_USER=theroot
WP_ADMIN_PASSWORD=123
WP_ADMIN_EMAIL=theroot@123.com
WP_USER=theuser
WP_PASSWORD=abc
WP_EMAIL=theuser@123.com
WP_ROLE=editor
```
4. Dowload the wordpress files.
```bash
sleep 10
wp --allow-root --path="/var/www/inception/" core download || true
```
5. If the wordpress files aren't there, create install wordpress and set it, if not move foward.
```bash
if ! wp --allow-root --path="/var/www/inception/" core is-installed;
then
    wp  --allow-root --path="/var/www/inception/" core install \
        --url=$WP_URL \
        --title=$WP_TITLE \
        --admin_user=$WP_ADMIN_USER \
        --admin_password=$WP_ADMIN_PASSWORD \
        --admin_email=$WP_ADMIN_EMAIL
fi;
```
6. Create the non-admin user and set its role.
```bash
if ! wp --allow-root --path="/var/www/inception/" user get $WP_USER;
then
    wp  --allow-root --path="/var/www/inception/" user create \
        $WP_USER \
        $WP_EMAIL \
        --user_pass=$WP_PASSWORD \
        --role=$WP_ROLE
fi;
```
7. Download another theme and activate it. It's not necessary, but I did because I don't like the default theme.
```bash
wp --allow-root --path="/var/www/inception/" theme install raft --activate 
```
8. start the php server in foreground.
```bash
exec $@
```
#### 1.2.5. Test the wordpress container
Go to the wordpress folder and run the following command .
```bash
docker build -t wordpress .
docker run -d wordpress
docker ps -a
docker exec -it copiedID /bin/bash
```
Now, you are inside the container. Run the following command to check if the wordpress files are there. The sleep here is used to give time to the container to download the files.
```bash
sleep 30 && ls /var/www/inception/
```
If you see the wordpress files, it means that all is ok. At the moment we won't check if the containers is talking with each other because we'll do it with the compose file. So, exits the container and let's clean our container test.
```bash
docker rm -f $(docker ps -aq) &&  docker rmi -f $(docker images -aq)
```
### 1.3. NGINX
Files: [Dockerfile](https://github.com/waltergcc/42-inception/blob/main/inception/srcs/requirements/nginx/Dockerfile), [conf/server.conf](https://github.com/waltergcc/42-inception/blob/main/inception/srcs/requirements/nginx/conf/server.conf), [conf/nginx.conf](https://github.com/waltergcc/42-inception/blob/main/inception/srcs/requirements/nginx/conf/nginx.conf)

#### 1.3.1. The Dockerfile
1. Use the debian 11 (bullseye) image
2. Indicates that this container will be listening on port 443
3. Update the system and install `nginx`, `openssl` only
4. Define the ARG to use in the next commands. At the final version, we'll have a .env file with all the variables, so you need to remove these declaration lines and keep the line that call all variables. 
```Dockerfile
ARG	CERT_FOLDER=/etc/nginx/certs/
ARG	CERTIFICATE=/etc/nginx/certs/certificate.crt
ARG	KEY=/etc/nginx/certs/certificate.key
ARG	COUNTRY=BR
ARG	STATE=BA
ARG	LOCALITY=Salvador
ARG	ORGANIZATION=42
ARG	UNIT=42
ARG	COMMON_NAME=login.42.fr
```
5. Create the folder for the certificates and generate these. It'll create a self-signed certificate that will be valid for 365 days. The `subj` flag is used to set the certificate information.
```Dockerfile
RUN	mkdir -p ${CERT_FOLDER} && \
	openssl req -newkey rsa:4096 -x509 -sha256 -days 365 -nodes \
	-out ${CERTIFICATE} \
	-keyout ${KEY} \
	-subj "/C=${COUNTRY}/ST=${STATE}/L=${LOCALITY}/O=${ORGANIZATION}/OU=${UNIT}/CN=${COMMON_NAME}"
```
6. Copy the configuration files to the container and complete the server.conf file with the variables that will be passed by the .env file.
```Dockerfile
COPY	conf/nginx.conf	/etc/nginx/
COPY	conf/server.conf	/etc/nginx/conf.d/

RUN	echo "\tserver name ${COMMON_NAME};\n\
	\tssl_certificate ${CERTIFICATE};\n\
	\tssl_certificate_key ${KEY};\n\
	}" >> /etc/nginx/conf.d/server.conf
```
7. Create the folder for the wordpress files and change its owner to `www-data` user.
```Dockerfile
RUN	mkdir -p /var/www/
RUN	chown -R www-data:www-data /var/www/
```
8. Start the nginx server in foreground.
```Dockerfile
CMD	["nginx", "-g", "daemon off;"]
```
#### 1.3.2. server.conf
The important thing here is change the set the port to 443 and ssl protocols to TLSv1.2 and the root folder to `/var/www/inception/`.
```conf
listen 443 ssl;
listen [::]:443 ssl;
ssl_protocols TLSv1.2;
root /var/www/inception/;
index index.php index.html;
```
At the end, the file has missing lines that will be completed in the Dockerfile. These are the information about the certificates that will be passed by the .env file. It's important that we never public files with confidential information.

#### 1.3.3. nginx.conf
The important thing here is change the user to `www-data` and set communication with php-fpm to port 9000 to use the wordpress files.
```conf
user www-data;
upstream php7.4-fpm
{
	server wordpress:9000;
}
```
#### 1.3.4. Test the nginx image
Go to the nginx folder and run the following command. This time we don't run the container because it need to connect with the wordpress container. And we'll do it with the compose file. We'll the built command only to check if the image is ok then we'll remove it.
```bash
docker build -t nginx .
docker images
docker rmi -f nginx
```
## 2. Docker-compose
Now that we have all Dockerfiles working well, we'll create the docker-compose file to run all containers together. Before we start, we need to create setup the docker-compose plugin. 

Check if the plugin is already installed with the command:
```bash
docker compose version
```
If it's not installed, run the following command:
```bash
sudo apt-get install docker-compose-plugin
```
Files: [requirements](https://github.com/waltergcc/42-inception/tree/main/inception/srcs/requirements), [docker-compose.yml](https://github.com/waltergcc/42-inception/blob/main/inception/srcs/docker-compose.yml), .env

Now you'll create a folder called `requirements` and inside it put all the folders that we created before. Then, create a file called `docker-compose.yml` and a file called `.env`.

### 2.1. docker-compose.yml
Start it with the following line to start the services definition.
```yml
services:
```
Then define the mariadb service. It's field are self-explanatory. Build is where the Dockerfile is, volumes is where the database files will be saved in the container, networks is the network that the container will use, init is used to run the setup.sh script, restart is used to restart the container if it fails, and env_file is the file that contains the variables that will be used in the container.
```yml
  mariadb:
    container_name: mariadb
    build: ./requirements/mariadb/
    volumes:
      - database:/var/lib/mysql/
    networks:
      - all
    init: true
    restart: on-failure
    env_file:
      - .env
```
The wordpress service is similar to the mariadb service, but it has a depends_on field that indicates that the wordpress container will only start after the mariadb container is running and volume and the build path are different.
```yml
  wordpress:
    container_name: wordpress
    build: ./requirements/wordpress/
    volumes:
      - wordpress_files:/var/www/inception/
    networks:
      - all
    init: true
    restart: on-failure
    env_file:
      - .env
    depends_on:
      - mariadb
```
the NGINX service depends on the wordpress service and has a ports field that indicates that the container will be listening on port 443. The build field beyound the path, it has some arguments that will be used in the Dockerfile given by the .env file.
```yml
nginx:
    container_name: nginx
    build:
      context: ./requirements/nginx/
      args:
        CERT_FOLDER: ${CERT_FOLDER}
        CERTIFICATE: ${CERTIFICATE}
        KEY: ${KEY}
        COUNTRY: ${COUNTRY}
        STATE: ${STATE}
        LOCALITY: ${LOCALITY}
        ORGANIZATION: ${ORGANIZATION}
        UNIT: ${UNIT}
        COMMON_NAME: ${COMMON_NAME}
    ports:
      - '443:443'
    volumes:
      - wordpress_files:/var/www/inception/
    networks:
      - all
    init: true
    restart: on-failure
    env_file:
      - .env
    depends_on:
      - wordpress
```

The volumes define the local host folder that will be used to save the database and the wordpress files. The subject informs that the data must be in user home directory. This volumes will work like a shared folder between the host and the containers.
```yml
volumes:

  database:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: ~/data/database

  wordpress_files:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: ~/data/wordpress_files
```
The networks define the network that the containers will use to communicate with each other. This is like a virtual switch that will connect the containers. 
```yml
networks:
  all:
    driver: bridge
```
### 2.2. .env
In this file, we'll put all the variables that we'll use in the docker-compose file. It's important that we never public files with confidential information. You'll put your .env file in the repo only in the evaluation time. Don't forget to remove the test variables that we used before in `mariadb/tools/setup.sh`, `wordpress/conf/wp-config.php`, `wordpress/tools/setup.sh` and `nginx/Dockerfile`. 

In the nginx Dockerfile we'll keep this line:
```Dockerfile
ARG	CERT_FOLDER CERTIFICATE KEY COUNTRY STATE LOCALITY ORGANIZATION UNIT COMMON_NAME
```
Now, the .env file will have the following variables:

```bash
# Database settings
DB_NAME=XXXXXXXX
DB_USER=XXXXXXXX
DB_PASSWORD=XXXXXXXX
DB_HOST=XXXXXXXX
DB_PASS_ROOT=XXXXXXXX

# Wordpress settings
WP_URL=XXXXXXXX
WP_TITLE=XXXXXXXX
WP_ADMIN_USER=XXXXXXXX
WP_ADMIN_PASSWORD=XXXXXXXX
WP_ADMIN_EMAIL=XXXXXXXX
WP_USER=XXXXXXXX
WP_PASSWORD=XXXXXXXX
WP_EMAIL=XXXXXXXX
WP_ROLE=XXXXXXXX
WP_FULL_URL=XXXXXXXX

# SSL settings
CERT_FOLDER=XXXXXXXX
CERTIFICATE=XXXXXXXX
KEY=XXXXXXXX
COUNTRY=XXXXXXXX
STATE=XXXXXXXX
LOCALITY=XXXXXXXX
ORGANIZATION=XXXXXXXX
UNIT=XXXXXXXX
COMMON_NAME=XXXXXXXX
```
### 2.3. docker-compose test
Now, all is setup correctly, so we can start the containers together. Go to the folder that contains the docker-compose.yml file and run the following command:
```bash
docker compose up
```
If all is ok, you'll see the containers running and the terminal won't return the prompt. Now to test, go to your browser and type the following address:
```bash
https://localhost
```
If all is ok, you can see the wordpress page. We're close to the end, but we need to do some more things. Let's clean our test. First, stop the compose with `ctrl + c` then run the following command to clean the containers, images, volumes and networks.
```bash
docker stop $(docker ps -qa) && \
	docker rm -f $(docker ps -qa) && \
	docker rmi -f $(docker images -qa) && \
	docker volume rm $(docker volume ls -q) && \
	docker network rm $(docker network ls -q) 2> /dev/null
```
## 3. The Makefile
Files: [srcs](https://github.com/waltergcc/42-inception/tree/main/inception/srcs), 
 [Makefile](https://github.com/waltergcc/42-inception/blob/main/inception/Makefile)

Before all, create a folder called `srcs` and put all files that we created before inside it.

Install the `hostsed` package to make easy the way to put our host url in the `/etc/hosts` file. Run the following command:
```bash
sudo apt-get install hostsed
```
Now, we'll create a Makefile to run the docker-compose commands. In my Makefile I've implemented some more commands, but in a simple way, it must have at lest two rules: `up` and `down`.
```Makefile
NAME		= inception
SRCS		= ./srcs
COMPOSE		= $(SRCS)/docker-compose.yml
HOST_URL	= login.42.fr
```
The up rule will create the folders that will be used to save the database and the wordpress files, add the host url to the `/etc/hosts` file, then run the docker-compose command.
```Makefile
up:
	mkdir -p ~/data/database
	mkdir -p ~/data/wordpress_files
	sudo hostsed add 127.0.0.1 $(HOST_URL)
	docker compose -p $(NAME) -f $(COMPOSE) up --build || (echo " $(FAIL)" && exit 1)
```
The down rule will remove the host url from the `/etc/hosts` file, then run the docker-compose command to stop the containers.
```Makefile
	sudo hostsed rm 127.0.0.1 $(HOST_URL)
	docker compose -p $(NAME) down
```
After that, if will want, you can create more rules. But for now. Our Makefile is ready to use, so in its folder, run `make up`, so you can go to your browser and type the following address:
```bash
https://login.42.fr
```
If all is ok, you can see the wordpress page. Now, to stop the containers, press in the running terminal `Ctrl + C` then run `make down`.

Now, Our needed files are ready. So we can go to the VM setup.

## 4. The VM
This stage will have a lot of steps, but after that you're be able to deploy your website in your VM, so let's go.

### 4.1. VM creation
1. Download debian image. I prefer to use the Debian 11. Follow the link:
https://cdimage.debian.org/cdimage/archive/11.7.0/amd64/iso-cd/debian-11.7.0-amd64-netinst.iso
2. Open the VirtualBox and create a new VM as Linux Debian 64 bits.
3. Set the RAM to 4096 MB
4. Create a dynamic VDI with at least 30 GB
5. Go to the VM settings > System > Motherboard and set the boot order to Optical, Hard Disk, Network.
6. Then at processor tab, set the number of processors to 4.
7. In the display menu, set the video memory to 128 MB.
8. In the audio menu, disable the audio.
9. In the network menu, set the network to NAT.
10. In the storage, select the CD icon and select the debian image that you downloaded.
11. Now start your VM.

### 4.2. Debian installation
1. Select install
2. then follow the normal installation steps, choosing region, user, password, etc. Nothing special here.
3. In the partition menu, select the guided - use entire disk - LVM
4. After that, select separate var/ tmp/ home/ partitions and Confirm it.
5. In the software selection, select only XFCE, Webserver, SSH server and standard system utilities.
6. In the GRUB menu, select yes and select the disk that you created.
7. At the end, your VM will reboot with the debian installed.

### 4.3. VM setup

#### 4.3.1. Add user as Sudo
Access as root and add the user to the sudo group.
```bash
su -
usermod -aG sudo user
```
After that, add the user into sudoers file.
```bash
sudo visudo
```
Then add the following line in the end of the file and save it.
```bash
user ALL=(ALL) ALL
```
Now, reboot the VM.

#### 4.3.2. Enable the Shared folder
1. In your main PC, create a folder in your home directory called `shared` . This folder will be used to share files between your main PC and the VM.
2. In the VirtualBox settings > Shared Folders, add a new shared folder with the name `shared` and the path to the folder that you created in your main PC and check the auto-mount and make permanent options.
3. Now, in the VM, at the VirtualBox menu > Devices > select insert Guest Additions CD image.
4. Open the terminal in the CD folder and run the following command.
```bash
sudo sh VBoxLinuxAdditions.run
sudo reboot
```
5. add your user to the vboxsf group and define your user as owner of the shared folder.
```bash
sudo usermod -a -G vboxsf your_user
sudo chown -R your_user:users /media/
```
6. Logout and login again to apply the changes. Now, you can see the shared folder in the `/media` folder as a external device.
7. (Optional) If you want to enable the copy and paste between the VM and your main PC, go to the VM menu > Devices > Shared Clipboard > Bidirectional. With this option, you can copy and paste text between the VM and your main PC.

#### 4.3.3. Install Docker and docker-compose
Prepare the docker repository installation
```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```
Then install docker and plugins
```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
Now add your user to the docker group. It's important use the docker commands without sudo.
```bash
sudo usermod -aG docker your_user
su - your_user
sudo reboot
```
Now check if the docker is working well with the following command:
```bash
docker run hello-world
```
#### 4.3.4. Install make and hostsed
```bash
sudo apt-get install -y make hostsed
```
## 5. The Website

### 5.1. Add your files in the VM
1. Copy your repo link in your main PC and go to the shared folder. Then clone it or copy and paste your files in the shared folder.
2. Copy your confidential .env file from your main PC to the VM. Paste it in folder srcs, inside the shared folder.

### 5.2. Start the containers
1. Go to the root of your project and run `make up`
2. Go to your browser and type the following address:
```bash
https://login.42.fr
```
If all is ok, you can see the wordpress page.
### 5.3. Credentials check
1. Try to input the follow link in your browser. If all is ok, you can't connect to the database because you'll try to connect with wrong port.
```bash
http://login.42.fr
```
2. Go back to right link, click in the lock icon in the left of the address bar and click in the certificate option to see the certificate information.
3. Now, enter in your browser with the following link to acess the wordpress admin page. Try to login with the admin user and the user. If all is ok, you can see the admin page dashboard.
```bash
https://login.42.fr/wp-admin
```

### 5.4. mariaDB check
Open another terminal and keep the terminal with the compose running. On this other terminal, run the following command to enter in the mariadb container.
```bash
docker exec -it mariadb /bin/bash
```
Then run the command to enter in the mysql
```bash
mysql -u your_user -p db_name
```
Then run the command to see the tables
```bash
SHOW TABLES;
```
If you see the tables, it means that all is ok. If you want to see the database, run the following command:
```bash
SELECT * FROM table_name\G;
```
And if you want to delete a row in a table, run the following command:
```bash
DELETE FROM table_name WHERE column_name = some_value;
```
After that, you can exit the mysql and the container and all your project work is done. Congratulations!