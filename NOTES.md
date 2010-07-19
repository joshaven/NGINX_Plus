# Introduction
My How-To LNMP (build instructions)

Note: The following code must not be pasted with extra spaces in front of it... for example 
"ENDCONFIG" is entirely different from "    ENDCONFIG".  The content of this file is formatted 
for markdown, if you are looking at a raw copy of this file then you will have to remove spacing.  
I suggest copying this text from the webpage and if all goes well... the spacing will be correct. 
You would do well to paste in a text editor before pasting in your server's console.

# Build and Install Nginx (newest) deb repo is too old to work with php

    # Everyhting here, unless stated otherwise, is ment to be run as a user other then root that has sudo privileges
    mkdir -p ~/src
    sudo apt-get install gcc make libpcre3-dev libssl-dev libsha-ocaml-dev

    sudo mkdir -p /etc/nginx/conf.d
    sudo mkdir -p /etc/nginx/sites-available
    sudo mkdir -p /etc/nginx/sites-enabled
    sudo mkdir -p /var/lib/nginx/body
    sudo mkdir -p /var/lib/nginx/proxy
    sudo mkdir -p /var/lib/nginx/fastcgi
    sudo mkdir -p /var/www

    cd ~/src
    wget http://nginx.org/download/nginx-0.7.67.tar.gz
    tar zxf ng* && cd ng*

    ./configure --prefix=/usr \
    --sbin-path=/usr/sbin/nginx \
    --conf-path=/etc/nginx/nginx.conf \
    --lock-path=/var/lock/nginx.lock \
    --error-log-path=/var/log/nginx/error.log \
    --http-log-path=/var/log/nginx/access.log \
    --pid-path=/var/run/nginx.pid \
    --user=www-data \
    --group=www-data \
    --http-client-body-temp-path=/var/lib/nginx/body \
    --http-proxy-temp-path=/var/lib/nginx/proxy \
    --http-fastcgi-temp-path=/var/lib/nginx/fastcgi \
    --with-md5=/usr/lib \
    --with-sha1=/usr/lib \
    --with-http_ssl_module \
    --with-http_gzip_static_module \
    --with-pcre \
    --without-mail_pop3_module \
    --without-mail_imap_module \
    --without-mail_smtp_module \
    --with-http_stub_status_module
    
    make
    sudo make install

# Configure Nginx
## Create startup script in /etc/init.d/
### Ensure file exists and make user writable
    sudo touch /etc/init.d/nginx
    sudo cp /etc/init.d/nginx /tmp/init.d_nginx.backup
### Write to file    
    sudo tee /etc/init.d/nginx <<ENDCONFIG
    #! /bin/sh

    ### BEGIN INIT INFO
    # Provides:          nginx
    # Required-Start:    \$all
    # Required-Stop:     \$all
    # Default-Start:     2 3 4 5
    # Default-Stop:      0 1 6
    # Short-Description: starts the nginx web server
    # Description:       starts nginx using start-stop-daemon
    ### END INIT INFO

    PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
    DAEMON=/usr/sbin/nginx
    NAME=nginx
    DESC=nginx

    test -x \$DAEMON || exit 0

    # Include nginx defaults if available

    if [ -f /etc/default/nginx ] ; then
            . /etc/default/nginx
    fi

    set -e

    case "\$1" in
      start)
            echo -n "Starting \$DESC: "
            start-stop-daemon --start --quiet --pidfile /var/run/\$NAME.pid \
                    --exec \$DAEMON -- \$DAEMON_OPTS || true
            echo "\$NAME."
            ;;
      stop)
            echo -n "Stopping \$DESC: "
            start-stop-daemon --stop --quiet --pidfile /var/run/\$NAME.pid \
                    --exec \$DAEMON || true
            echo "\$NAME."
            ;;
      restart|force-reload)
            echo -n "Restarting \$DESC: "
            start-stop-daemon --stop --quiet --pidfile \
                    /var/run/\$NAME.pid --exec \$DAEMON || true
            sleep 1
            start-stop-daemon --start --quiet --pidfile \
                    /var/run/\$NAME.pid --exec \$DAEMON -- \$DAEMON_OPTS || true
            echo "\$NAME."
            ;;
      reload)
          echo -n "Reloading \$DESC configuration: "
          start-stop-daemon --stop --signal HUP --quiet --pidfile /var/run/\$NAME.pid \
              --exec \$DAEMON || true
          echo "\$NAME."
          ;;
      *)
            N=/etc/init.d/\$NAME
            echo "Usage: \$N {start|stop|restart|reload|force-reload}" >&2
            exit 1
            ;;
    esac

    exit 0
    ENDCONFIG

### Tell server to start Nginx as a service
    sudo update-rc.d nginx defaults


## Create Nginx Config file

### Ensure file exists & set to user writable
    sudo touch /etc/nginx/nginx.conf
    sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.backup

### Write nginx config file    
    sudo tee /etc/nginx/nginx.conf <<ENDCONFIG
    user www-data;
    worker_processes  5;

    error_log  /var/log/nginx/error.log;
    pid        /var/run/nginx.pid;

    events {
        worker_connections  1024;
    }

    http {
        server_tokens off;
        include       /etc/nginx/mime.types;
        default_type  application/octet-stream;

        access_log  /var/log/nginx/access.log;

        sendfile        on;
        #tcp_nopush     on;

        #keepalive_timeout  0;
        keepalive_timeout  65;
        tcp_nodelay        on;
        gzip  on;

        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;
    }
    ENDCONFIG


## Ensure default site config exists and is setup for PHP
### make user writeable... this is so the following can be run as a non-privileged user

    sudo touch /etc/nginx/sites-available/default
    sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/default.backup
    
### Write default site config
    sudo tee /etc/nginx/sites-available/default <<ENDCONFIG
    server {
        listen   80;
        server_name  localhost;
        access_log  /var/log/nginx/localhost.access.log;

    ## Default location
        location / {
            root   /var/www;
            index  index.php;
        }

    ## Images and static content is treated different
        location ~* ^.+.(jpg|jpeg|gif|css|png|js|ico|xml)\$ {
          access_log        off;
          expires           30d;
          root /var/www;
        }

    ## Parse all .php file in the /var/www directory
        location ~ .php\$ {
            fastcgi_split_path_info ^(.+\.php)(.*)\$;
            fastcgi_pass   backend;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  /var/www\$fastcgi_script_name;
            include fastcgi_params;
            fastcgi_param  QUERY_STRING     \$query_string;
            fastcgi_param  REQUEST_METHOD   \$request_method;
            fastcgi_param  CONTENT_TYPE     \$content_type;
            fastcgi_param  CONTENT_LENGTH   \$content_length;
            fastcgi_intercept_errors        on;
            fastcgi_ignore_client_abort     off;
            fastcgi_connect_timeout 60;
            fastcgi_send_timeout 180;
            fastcgi_read_timeout 180;
            fastcgi_buffer_size 128k;
            fastcgi_buffers 4 256k;
            fastcgi_busy_buffers_size 256k;
            fastcgi_temp_file_write_size 256k;
        }

    ## Disable viewing .htaccess & .htpassword
        location ~ /\.ht {
            deny  all;
        }
    }
    upstream backend {
            server 127.0.0.1:9000;
    }
    ENDCONFIG

### Enable default site

    sudo ln -s /etc/nginx/sites-available/default /etc/nginx/sites-enabled/default

### Create Default site

    sudo touch /var/www/index.php
    echo '<?php phpinfo(); ?>' | tee /var/www/index.php
    sudo chmod 444 /var/www/index.php
    sudo chown www-data /var/www/index.php
    sudo chgrp www-data /var/www/index.php



# Install PHP

    sudo apt-get update
    sudo apt-get installphp5-common php5-dev php5-fpm php-pear php5-cli php5-suhosin 

## Add dotdeb Repository
### Add dotdeb Repository to sources
    sudo cp /etc/apt/sources.list /etc/apt/sources.list.original
    echo "########################################################################" | sudo tee -a /etc/apt/sources.list
    echo "# php53.dotdeb.org Repository (Easier installation & maintenance of PHP)" | sudo tee -a /etc/apt/sources.list
    echo "deb http://php53.dotdeb.org stable all" | sudo tee -a /etc/apt/sources.list

### Import the signatures for the dotdeb repo

    wget http://packages.dotdeb.org/dotdeb.gpg && sudo apt-key add dotdeb.gpg && rm dotdeb.gpg
    sudo apt-get update
    
### Add memcache for PHP (distributed cache)
    sudo apt-get install memcached php5-memcache && sudo /etc/init.d/php5-fpm restart
    
    # Confirm memcache is working (you'll get a not present message if it is not)
    php --ri memcache
    
    # If you need to remove memcache then do:
    #   sudo apt-get -y remove memcached php5-memcache && sudo apt-get -y autoremove 
    #   sudo dpkg --purge $(dpkg --list|awk '{print $2}'|grep memcache)
    #   sudo /etc/init.d/php5-fpm restart
    
## Install APC for PHP (non-distributed cache)

    sudo apt-get install php5-apc && sudo /etc/init.d/php5-fpm restart
    
    # Confirm apc is working (you'll get a not present message if it is not)
    php --ri apc
    
    # If you need to remove APC then do:
    #   sudo apt-get -y remove php5-apc && sudo apt-get -y autoremove 
    #   sudo dpkg --purge $(dpkg --list|awk '{print $2}'|grep php5-apc)
    #   sudo /etc/init.d/php5-fpm restart
    
# Install MySQL & SQLite

    sudo apt-get install mysql-server php5-mysql php5-sqlite
    
    php --ri mysql
    php --ri sqlite