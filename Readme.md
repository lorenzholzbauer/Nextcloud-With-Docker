# How to set up Nextcloud in a Docker container
## Prerequisits
- A Linux server (self-hosted or a virtual server)
- Docker CE and Docker compose
- A domain

I would recommend setting up docker on a non-root account but it will work just fine with root access

## Create a setup directory
```mkdir ~/nextcloud && cd ~/nextcloud```

## Create the Docker compose
```nano docker-compose.yml```

And paste the following
```
version: '3'

volumes:
  nextcloud:
  db:

services:
  db:
    image: mariadb
    restart: always
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW --innodb-file-per-table=1 --skip-innodb-read-only-compressed
    volumes:
      - db:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=placeholderrootsqlpwd
      - MYSQL_PASSWORD=placeholdersqlpwd
      - MYSQL_DATABASE=db
      - MYSQL_USER=placeholderuser

  redis:
    image: redis
    restart: always
    command: redis-server --requirepass placeholderredispwd

  app:
    image: nextcloud
    restart: always
    ports:
      - 8080:80
    links:
      - db
      - redis
    volumes:
      - nextcloud:/var/www/html
    environment:
      - MYSQL_PASSWORD=placeholdersqlpwd
      - MYSQL_DATABASE=db
      - MYSQL_USER=placeholderuser
      - MYSQL_HOST=db
      - REDIS_HOST_PASSWORD=placeholderredispwd
    depends_on:
      - db
      - redis
```

If you want you can change the placeholder passwords but it's no necessity i guess

## Run a docker compose file
```docker-compose up -d```

Check if everything is running with: 

```docker-compose ps```

## Setup nginx
If you haven't already set up a https certificate with letsencrypt:

https://eff-certbot.readthedocs.io/en/stable/using.html#standalone

Remember to stop the nginx service with ```systemctl stop nginx.service``` and to point your domain to the server

Install nginx with the package manager of your choice

```nano /etc/nginx/sites-available/nextcloud.conf```

And paste the following

```
server {
    if ($host = nextcloud.yourdomain.tld) {
        return 301 https://$host$request_uri;
    }
        listen 80;
        listen [::]:80;

        server_name nextcloud.yourdomain.tld;
    return 404;
}

# server block for HTTPS secure connections
server {
    listen [::]:443 ssl http2;
    listen 443 ssl http2;

    ssl_certificate /etc/letsencrypt/live/nextcloud.yourdomain.tld/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/nextcloud.yourdomain.tld/privkey.pem;

    server_name nextcloud.yourdomain.tld;
    root /var/www/html;

    index index.html;

    location / {
        proxy_pass http://localhost:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload";
        client_max_body_size 0;

        access_log /var/log/nginx/nextcloud.access.log;
        error_log /var/log/nginx/nextcloud.error.log;
    }

    location /.well-known/carddav {
      return 301 $scheme://$host/remote.php/dav;
    }

    location /.well-known/caldav {
      return 301 $scheme://$host/remote.php/dav;
    }
}
```

Remember to change "nextcloud.yourdomain.tld" to your own domain

Now link the config to nginx

```ln -s /etc/nginx/sites-available/nextcloud.conf /etc/nginx/sites-enabled/```

Use this command to verify the config

```nginx -t```

And finally restart nginx

```sudo systemctl restart nginx```

## Setup Nextcloud
Now open nextcloud with the domain you configured and enjoy