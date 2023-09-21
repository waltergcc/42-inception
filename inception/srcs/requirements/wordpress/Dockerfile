
# this dockerfile is used to build the wordpress container
# the EXPOSE port is 9000 its a way to tell docker that the container will use this port
# it install ca-certificates, php7.4-fpm, php7.4-mysql, wget, tar and copy the configuration files
# the RUN sed changes the php.ini file to fix the path info
# RUN wget downloads the wp-cli.phar file that is used to install wordpress through the command line
# at the end creates the folders and set the permissions to the www-data user
# the runs the setup.sh script that is used to install wordpress
# and start the php-fpm7.4 server in the foreground

FROM	debian:bullseye

EXPOSE	9000

ARG		PHPPATH=/etc/php/7.4/fpm

RUN		apt update && apt install --no-install-recommends --no-install-suggests -y \
		ca-certificates \
		php7.4-fpm \
		php7.4-mysql \
		wget \
		tar && \
		rm -rf /var/lib/apt/lists/*

RUN		service php7.4-fpm stop

COPY	conf/www.conf ${PHPPATH}/pool.d/

RUN		sed -i 's/;cgi.fix_pathinfo=1/cgi.fix_pathinfo=0/g' ${PHPPATH}/php.ini && \
		sed -i "s/listen = \/run\/php\/php$PHP_VERSION_ENV-fpm.sock/listen = 9000/g" ${PHPPATH}/pool.d/www.conf && \
		sed -i 's/;listen.mode = 0660/listen.mode = 0660/g' ${PHPPATH}/pool.d/www.conf && \
		sed -i 's/;daemonize = yes/daemonize = no/g' ${PHPPATH}/pool.d/www.conf

RUN		wget --no-check-certificate https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar && \
		chmod +x wp-cli.phar && \
		mv wp-cli.phar /usr/local/bin/wp

RUN		mkdir -p /run/php/ && \
		mkdir -p /var/run/php/ && \
		mkdir -p /var/www/inception/

RUN		chown -R www-data:www-data /var/www/inception/

COPY	conf/wp-config.php /tmp/

COPY	tools/setup.sh /bin/
RUN		chmod +x /bin/setup.sh

CMD		["setup.sh", "php-fpm7.4", "--nodaemonize"]