# WP-FAQ
### Links:
- [docker installation](https://docs.docker.com/engine/install/)
- [vsftpd installation tutorial](https://phoenixnap.com/kb/install-ftp-server-on-ubuntu-vsftpd)
- [DO WP(in docker) tutorial](https://www.digitalocean.com/community/tutorials/how-to-install-wordpress-with-docker-compose)
- [DO WP(on host) tutorial](https://www.digitalocean.com/community/tutorials/how-to-install-wordpress-with-lemp-on-ubuntu-18-04)
### docker-compose file example:
```
version: '3'
services:
###***** MySQL *****###
  db:
    image: mysql:8.0
    container_name: db
    restart: unless-stopped
    env_file: .env
    environment:
      - MYSQL_DATABASE=$DB_NAME
    volumes: 
      - ./dbdata:/var/lib/mysql
    command: '--default-authentication-plugin=mysql_native_password'
    networks:
      - app-network
###***** Wordpress *****###
  wordpress:
    depends_on: 
      - db
    container_name: wordpress
    image: wordpress:5.4.2-php7.4-fpm
    restart: unless-stopped
    env_file: .env
    environment:
      - WORDPRESS_DB_HOST=db:3306
      - WORDPRESS_DB_USER=$MYSQL_USER
      - WORDPRESS_DB_PASSWORD=$MYSQL_PASSWORD
      - WORDPRESS_DB_NAME=$DB_NAME
    volumes:
      - ./wordpress:/var/www/html
    networks:
      - app-network
###***** Nginx *****###
  webserver:
    depends_on:
      - wordpress
    image: nginx:1.15.12-alpine
    container_name: webserver
    restart: unless-stopped
    command: "/bin/sh -c 'while :; do sleep 6h & wait $${!}; nginx -s reload; done & nginx -g \"daemon off;\"'"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./wordpress:/var/www/html
      - ./nginx-conf:/etc/nginx/conf.d   
      - ./data/certbot/conf:/etc/letsencrypt
      - ./data/certbot/www:/var/www/certbot
    networks:
      - app-network

###*****Certbot*****### uncomment if https needed 
#  certbot:
#    container_name: Certbot
#    image: certbot/certbot
#    volumes:
#      - ./data/certbot/conf:/etc/letsencrypt
#      - ./data/certbot/www:/var/www/certbot
#    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"

volumes:
  wordpress:
  dbdata:

networks:
  app-network:
    driver: bridge
```
### .env file example:
```
MYSQL_ROOT_PASSWORD=
MYSQL_USER=
DB_NAME=
MYSQL_PASSWORD=
```
### Default Nginx config
```
mkdir nginx-conf
nano nginx-conf/nginx.conf
Past this config:
server {
        listen 80;
        listen [::]:80;

        server_name example.com www.example.com;

        index index.php index.html index.htm;

        root /var/www/html;

        location ~ /.well-known/acme-challenge {
                allow all;
                root /var/www/html;
        }

        location / {
                try_files $uri $uri/ /index.php$is_args$args;
        }

        location ~ \.php$ {
                try_files $uri =404;
                fastcgi_split_path_info ^(.+\.php)(/.+)$;
                fastcgi_pass wordpress:9000;
                fastcgi_index index.php;
                include fastcgi_params;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                fastcgi_param PATH_INFO $fastcgi_path_info;
        }

        location ~ /\.ht {
                deny all;
        }

        location = /favicon.ico {
                log_not_found off; access_log off;
        }
        location = /robots.txt {
                log_not_found off; access_log off; allow all;
        }
        location ~* \.(css|gif|ico|jpeg|jpg|js|png)$ {
                expires max;
                log_not_found off;
        }
}
```
### Importantly:
- Run docker from user "www-data"
- Set 755 permission to WP-directory
- Configure FTP server(in some cases cant install plugins witout ftp)
### Run docker-compose:
```sh
docker-compose up -d
```

### How to setup&configure FTP:
```sh
$ sudo apt install vsftpd
$ sudo systemctl start vsftpd
$ sudo systemctl enable vsftpd
$ sudo systemctl restart vsftpd.service
test connection to frp server: $ ftp -p ftp_server_ip_adderss
```
#### Example main config file /etc/vsftpd.conf:
```
listen=YES
listen_ipv6=NO
anonymous_enable=NO
local_enable=YES
write_enable=YES
user_sub_token=$USER
local_root=/home/$USER/path_to_wp_project
pasv_enable=Yes
pasv_min_port=64000
pasv_max_port=64321
file_open_mode=0777
local_umask=022
pasv_address=server_ip_address
userlist_enable=YES
userlist_deny=NO
allow_writeable_chroot=YES
userlist_file=/etc/vsftpd.userlist
secure_chroot_dir=/var/run/vsftpd/empty
rsa_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
rsa_private_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
ssl_enable=NO
```
#### Example main config file /etc/vsftpd.userlist:
```
www-data
2nd-ftp_user
3th-ftp_user
```
### How to change main site url in mysql db:
```sh
$ mysql -u root -p
mysql> use wordpress;
mysql> show tables;
mysql> UPDATE `wp_options` SET option_value = ‘https://site.com’ WHERE option_name IN (‘siteurl’, ‘home’);
mysql> exit;
```
