# Use NGINX as a proxy to reach the containers

Install NGINX Open Source and Let's Encrypt on your visualization host (where your InfluxDB and Grafana containers are).

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

Inform SELinux that nginx can connect to localhost.

```bash
sudo setsebool -P httpd_can_network_connect 1
```

Start Nginx.

```bash
sudo systemctl nginx restart
```

## Done

Point your web browser to https://[remote_host]/grafana and configure the graphs to taste.
