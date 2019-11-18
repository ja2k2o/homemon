# Setting up database and data visualisation

These steps are performed _not_ on the Raspberry but on a remote host. This way we can store the measurements independently of the Raspberry (the SD card storage will crap itself sooner or later, SD is not designed to withstand constant rewrites) and also security is better as no users will use the same host for collecting the data and visualising it; you do not need to let outsiders (if you wish) to login to your home — to your Raspberry.

The remote host is assumed to be a virtual machine running CentOS 7.

## Docker

Install Docker.

```bash
sudo yum install docker docker-compose
```

## Compose Docker

Create `docker-compose.yml` (replace obvious parts with relevant stuff).

```bash
mkdir housemon && cd housemon
cat <<EOF > docker-compose.yml
version: '3'

services:
  influxdb:
    image: influxdb:latest
    container_name: influxdb
    restart: always
    ports:
      - "127.0.0.1:8086:8086"
    networks:
      - housemon-nw
    volumes:
      - influxdb-volume:/var/lib/influxdb
    environment:
      - INFLUXDB_REPORTING_DISABLED=true
      - INFLUXDB_HTTP_ENABLED=true
      - INFLUXDB_HTTP_AUTH_ENABLED=true

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: always
    ports:
      - "127.0.0.1:3000:3000"
    networks:
      - housemon-nw
    volumes:
      - grafana-volume:/var/lib/grafana
    environment:
      - GF_DEFAULT_INSTANCE_NAME=[hostname]
      - GF_SERVER_PROTOCOL=http
      - GF_SERVER_DOMAIN=[fully_qualified_domain_name_of_your_host]
      - GF_SERVER_ROOT_URL=https://%(domain)s/grafana/
      - GF_SECURITY_ADMIN_PASSWORD=[a_very_good_password] # comment this out after first run!
      - GF_AUTH_GOOGLE_ENABLED=true
      - GF_AUTH_GOOGLE_CLIENT_ID=[obtain_from_google]
      - GF_AUTH_GOOGLE_CLIENT_SECRET=[obtain_from_google]
      - GF_AUTH_GOOGLE_SCOPES=email profile
      - GF_AUTH_GOOGLE_AUTH_URL=https://accounts.google.com/o/oauth2/auth
      - GF_AUTH_GOOGLE_TOKEN_URL=https://accounts.google.com/o/oauth2/token
      - GF_AUTH_GOOGLE_ALLOWED_DOMAINS=gmail.com
      - GF_AUTH_GOOGLE_ALLOW_SIGN_UP=false

networks:
  housemon-nw:

volumes:
  grafana-volume:
    external: true
  influxdb-volume:
    external: true
EOF
```

## Prepare Docker for running

Create volumes and networks.

```bash
docker network create housemon-nw
docker volume create grafana-volume
docker volume create influxdb-volume
```

Initialise the database. Note that values for `INFLUXDB_USER` and `INFLUXDB_USER_PASSWORD` are the same ones you defined in [Step 2](step2.md#build-and-configure-ruuvicollector), `ruuvi-collector.properties` file.

```bash
docker run --rm \
  -e INFLUXDB_DB=house -e INFLUXDB_ADMIN_ENABLED=true \
  -e INFLUXDB_ADMIN_USER=admin \
  -e INFLUXDB_ADMIN_PASSWORD=[supersecretpassword] \
  -e INFLUXDB_USER=ruuvi -e INFLUXDB_USER_PASSWORD=[secretpassword] \
  -v influxdb-volume:/var/lib/influxdb \
  influxdb /init-influxdb.sh
```

Time to fire it up.

```bash
docker-compose up -d
```

Both the visualiser (Grafana) and database (InfluxDB) now run in their own containers, and are able to talk to each other. But not with outside world.

## Put a proxy in front of the containers

Install Nginx and Let's Encrypt.

```bash
sudo yum install nginx certbot
```

Configure SSL.

```bash
sudo certbot --nginx
```

Edit `/etc/nginx/nginx.conf` to contain the following:

```bash
http {
    […]
    upstream grafana {
        server 127.0.0.1:3000;
    }
    upstream influxdb {
        server 127.0.0.1:8086;
    }
    […]
    server {
        location /grafana {
            rewrite /grafana(.*) $1 break;
            proxy_pass http://grafana;
        }
        location /influxdb {
            rewrite /influxdb(.*) $1 break;
            proxy_pass http://influxdb;
        }
    }
    […]
}
```

Inform SELinux that Nginx can connect to localhost.

```bash
sudo setsebool -P httpd_can_network_connect 1
```

Start Nginx.

```bash
sudo systemctl nginx restart
```

## Done

Point your web browser to https://[remote_host]/grafana and configure graphs to taste.
