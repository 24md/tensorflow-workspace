# Set Up Tensorflow Workspace on a Vultr Cloud GPU Instance

The following are the prerequisites and the steps to set up Tensorflow Workspace on a Vultr Cloud GPU instance.

## Prerequisites

* Deploy a [Vultr Cloud GPU instance](https://www.vultr.com/marketplace/apps/nvidia-docker/) using the `nvidia-docker` marketplace app.
* Map a subdomain to the instance using an `A` record for permanent deployment.

## Deploy Temporary Workspace

Verify GPU availability on host machine.

```console
# nvidia-smi
```

Verify GPU availability in a container.

```console
# docker run --gpus all nvidia/cuda:10.2-base nvidia-smi
```

Deploy a `tensorflow/tensorflow-gpu-jupyter` container.

```console
# docker run -it -p 8888:8888 --gpus all --rm tensorflow/tensorflow:latest-gpu-jupyter
```

Create a new notebook and test the GPU availability in the Tensorflow Module.

```python
import tensorflow as tf

tf.config.list_physical_devices()
```

## Deploy Permanent Workspace

Create and enter a new directory.

```console
# mkdir tensorflow-workspace
# cd tensorflow-workspace
```

Create a new `docker-compose` file.

```console
# nano docker-compose.yaml
```

Add the following contents to the file.

```yaml
services:

  jupyter:
    image: tensorflow/tensorflow:latest-gpu-jupyter
    restart: unless-stopped
    volumes:
      - "./notebooks:/tf/notebooks"
    deploy:
      resources:
        reservations:
          devices:
            - capabilities: [gpu]

  nginx:
    image: nginx
    restart: unless-stopped
    ports:
      - 80:80
      - 443:443
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/dhparam.pem:/etc/ssl/certs/dhparam.pem
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot


  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot
    command: certonly --webroot -w /var/www/certbot --force-renewal --email YOUR_EMAIL -d YOUR_HOSTNAME --agree-tos
```

Create a new directory and file for `nginx` configuration.

```console
# mkdir nginx
# nano nginx/nginx.conf
```

Add the following contents to the file.

```nginx
events {}

http {
    server_tokens off;
    charset utf-8;

    server {
        listen 80 default_server;
        server_name _;

        location ~ /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }
    }
}
```

Create a new `dhparam` file using `openssl`.

```console
# openssl dhparam -dsaparam -out nginx/dhparam.pem 4096
```

Run the `docker-compose` file.

```console
# docker-compose up -d
```

After few minutes, check if the SSL certificate was generated.

```console
# ls certbot/conf/live/YOUR_HOSTNAME/
```

Stop the `nginx` container.

```console
# docker-compose stop nginx
```

Swap the `nginx` configuration.

```console
# rm -f nginx/nginx.conf
# nano nginx.conf
```

Add the following contents to the file.

```nginx
events {}

http {
    server_tokens off;
    charset utf-8;

    map $http_upgrade $connection_upgrade {
        default upgrade;
        '' close;
    }

    server {
        listen 80 default_server;
        server_name _;

        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl http2;

        server_name YOUR_HOSTNAME;

        ssl_certificate     /etc/letsencrypt/live/YOUR_HOSTNAME/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/YOUR_HOSTNAME/privkey.pem;

        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers on;
        ssl_dhparam /etc/ssl/certs/dhparam.pem;
        ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
        ssl_session_timeout 1d;
        ssl_session_cache shared:SSL:50m;
        ssl_stapling on;
        ssl_stapling_verify on;
        add_header Strict-Transport-Security max-age=15768000;

        location / {
            proxy_pass http://jupyter:8888;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
            proxy_set_header X-Scheme $scheme;

            proxy_buffering off;
        }

        location ~ /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }
    }
}
```

Start the `nginx` container.

```console
# docker-compose start nginx
```

Verify the deployment by opening your hostname in the web browser.

Fetch the Jupyter token from logs.

```console
# docker-compose logs jupyter
```

Add a new entry in the `crontab`.

```console
# crontab -e
```

Add the following lines to the table.

```cron
0 5 1 */2 *  /usr/local/bin/docker-compose start -f /root/tensorflow-workspace/docker-compose.yml certbot
5 5 1 */2 *  /usr/local/bin/docker-compose restart -f /root/tensorflow-workspace/docker-compose.yml nginx
```

Exit the `cron` editor using <kbd>Esc</kbd> then <kbd>!wq</kbd> and <kbd>Enter</kbd>.
