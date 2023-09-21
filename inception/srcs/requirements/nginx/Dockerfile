
# this dockerfile is used to build the image for the nginx container
# the EXPOSE port is 443 its a way to tell docker that the container will use this port
# it install nginx and openssl, create a self-signed certificate and copy the configuration files
# the ARG call receive args from the docker-compose.yml file that is given by the .env file
# it is the only way that I could find to pass the args to the dockerfile
# the RUN echo completes the server.conf file with the private information 
# the penultimate RUN define the www-data user as the owner of the /var/www/
# at the end it runs starts the nginx server in the foreground

FROM	debian:bullseye

EXPOSE	443

RUN		apt update && apt install -y --no-install-recommends --no-install-suggests \
		nginx \
		openssl && \
		rm -rf /var/lib/apt/lists/*

# ARG		CERT_FOLDER=/etc/nginx/certs/
# ARG		CERTIFICATE=/etc/nginx/certs/certificate.crt
# ARG		KEY=/etc/nginx/certs/certificate.key
# ARG		COUNTRY=BR
# ARG		STATE=BA
# ARG		LOCALITY=Salvador
# ARG		ORGANIZATION=42
# ARG		UNIT=42
# ARG		COMMON_NAME=login.42.fr

ARG		CERT_FOLDER CERTIFICATE KEY COUNTRY STATE LOCALITY ORGANIZATION UNIT COMMON_NAME

RUN mkdir -p ${CERT_FOLDER} && \
	openssl req -newkey rsa:4096 -x509 -sha256 -days 365 -nodes \
    -out ${CERTIFICATE} \
    -keyout ${KEY} \
    -subj "/C=${COUNTRY}/ST=${STATE}/L=${LOCALITY}/O=${ORGANIZATION}/OU=${UNIT}/CN=${COMMON_NAME}"

COPY	conf/nginx.conf	/etc/nginx/
COPY	conf/server.conf	/etc/nginx/conf.d/

RUN		echo "\tserver name ${COMMON_NAME};\n\
			\tssl_certificate ${CERTIFICATE};\n\
			\tssl_certificate_key ${KEY};\n\
			}" >> /etc/nginx/conf.d/server.conf

RUN		mkdir -p /var/www/
RUN		chown -R www-data:www-data /var/www/

CMD		["nginx", "-g", "daemon off;"]