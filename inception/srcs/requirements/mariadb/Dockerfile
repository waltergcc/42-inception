
# this dockerfile is used to build the image for the mariadb container
# it installs mariadb-server only and copies the configuration file
# the EXPOSE port is 3306 its a way to tell docker that the container will use this port
# at the end it runs the setup.sh and start the mysql

FROM	debian:bullseye

EXPOSE	3306

RUN		apt update && \
		apt install -y --no-install-recommends --no-install-suggests \
		mariadb-server && \
		rm -rf /var/lib/apt/lists/*

COPY	conf/50-server.cnf /etc/mysql/mariadb.conf.d/

COPY	tools/setup.sh /bin/
RUN		chmod +x /bin/setup.sh

CMD		["setup.sh", "mysqld_safe"]