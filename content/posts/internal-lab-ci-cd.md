+++ 
draft = false
date = 2024-09-22T16:54:22+07:00
title = "Internal lab setup notes - Self-hosting a CI/CD system"
+++

Use [incus]({{< ref "incus-rhel-profile" >}}) to provision a container to host [Jenkins](https://jenkins.io) service:
```sh
# Use managed bridge network for demo/prototype on a Linux workstation
IPV4_ADDR=$(incus network get incusbr0 ipv4.address)
NET_MASK=$(echo ${IPV4_ADDR} | awk -F/ '{print $2}')
SUB_NET=$(echo ${IPV4_ADDR} | awk -F/ '{print $1}' | awk -F\. '{print $1"."$2"."$3}')

incus create images:rockylinux/9/cloud jenkins --profile rhel

# set static IP for the host
cat <<EOF | incus config set jenkins cloud-init.network-config -
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    eth0:
      dhcp4: false
      routes:
        - to: 0.0.0.0/0
          via:  ${SUB_NET}.1
      addresses:
        - ${SUB_NET}.11/${NET_MASK}
      nameservers:
        addresses:
          - ${SUB_NET}.2
EOF

incus start jenkins
# wait for cloud-init to be done
incus exec jenkins -- cloud-init status --wait
# Check cloud-init logs in case any failure
incus exec jenkins -- tail -f /var/log/cloud-init.log
incus exec jenkins -- tail -f /var/log/cloud-init-output.log
incus restart jenkins
incus shell jenkins
```

DNS hostname: `jenkins.lab.internal`.

```sh
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

```sh
dnf module install nginx:1.24
systemctl enable --now nginx.service
dnf install python3-certbot-nginx
certbot -n --nginx --agree-tos --email postmaster@lab.internal -d jenkins.lab.internal --server https://ca.lab.internal/acme/acme/directory
systemctl enable --now certbot-renew.timer
```

[https://www.jenkins.io/doc/book/installing/linux/#long-term-support-release-3](https://www.jenkins.io/doc/book/installing/linux/#long-term-support-release-3)

```sh
wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
dnf update
dnf install java-21-openjdk-headless
dnf install jenkins
systemctl daemon-reload
systemctl enable --now jenkins
```

[https://www.jenkins.io/doc/book/system-administration/reverse-proxy-configuration-with-jenkins/reverse-proxy-configuration-nginx/](https://www.jenkins.io/doc/book/system-administration/reverse-proxy-configuration-with-jenkins/reverse-proxy-configuration-nginx/)

`/etc/nginx/nginx.conf`
```
# For more information on configuration, see:
#   * Official English Documentation: http://nginx.org/en/docs/
#   * Official Russian Documentation: http://nginx.org/ru/docs/

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log notice;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    keepalive_timeout   65;
    types_hash_max_size 4096;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;

    upstream jenkins {
        keepalive 32; # keepalive connections
        server 127.0.0.1:8080; # jenkins ip and port
    }

    # Required for Jenkins websocket agents
    map $http_upgrade $connection_upgrade {
        default upgrade;
        '' close;
    }

    server {
        server_name  jenkins.lab.internal;
        root /var/cache/jenkins/war/;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        # pass through headers from Jenkins that Nginx considers invalid
        ignore_invalid_headers off;

        location ~ "^/static/[0-9a-fA-F]{8}\/(.*)$" {
            # rewrite all static files into requests to the root
            # E.g /static/12345678/css/something.css will become /css/something.css
            rewrite "^/static/[0-9a-fA-F]{8}\/(.*)" /$1 last;
        }

        location /userContent {
            # have nginx handle all the static requests to userContent folder
            # note : This is the $JENKINS_HOME dir
            root /var/lib/jenkins/;
            if (!-f $request_filename){
                # this file does not exist, might be a directory or a /**view** url
                rewrite (.*) /$1 last;
                break;
            }
            sendfile on;
        }

        location / {
            sendfile off;
            proxy_pass         http://jenkins;
            proxy_redirect     default;
            proxy_http_version 1.1;

            # Required for Jenkins websocket agents
            proxy_set_header   Connection        $connection_upgrade;
            proxy_set_header   Upgrade           $http_upgrade;

            proxy_set_header   Host              $http_host;
            proxy_set_header   X-Real-IP         $remote_addr;
            proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Proto $scheme;
            proxy_max_temp_file_size 0;

            #this is the maximum upload size
            client_max_body_size       10m;
            client_body_buffer_size    128k;

            proxy_connect_timeout      90;
            proxy_send_timeout         90;
            proxy_read_timeout         90;
            proxy_request_buffering    off; # Required for HTTP CLI commands
        }

    listen [::]:443 ssl ipv6only=on; # managed by Certbot
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/jenkins.lab.internal/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/jenkins.lab.internal/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}

# Settings for a TLS enabled server.
#
#    server {
#        listen       443 ssl http2;
#        listen       [::]:443 ssl http2;
#        server_name  _;
#        root         /usr/share/nginx/html;
#
#        ssl_certificate "/etc/pki/nginx/server.crt";
#        ssl_certificate_key "/etc/pki/nginx/private/server.key";
#        ssl_session_cache shared:SSL:1m;
#        ssl_session_timeout  10m;
#        ssl_ciphers PROFILE=SYSTEM;
#        ssl_prefer_server_ciphers on;
#
#        # Load configuration files for the default server block.
#        include /etc/nginx/default.d/*.conf;
#
#        error_page 404 /404.html;
#        location = /404.html {
#        }
#
#        error_page 500 502 503 504 /50x.html;
#        location = /50x.html {
#        }
#    }



    server {
    if ($host = jenkins.lab.internal) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


        listen       80;
        listen       [::]:80;
        server_name  jenkins.lab.internal;
    return 404; # managed by Certbot


}}
```

```sh
systemctl edit jenkins
```

`/etc/systemd/system/jenkins.service.d/override.conf`
```
[Service]
Environment=JENKINS_LISTEN_ADDRESS=127.0.0.1
Environment=JAVA_OPTS="-Djenkins.branch.WorkspaceLocatorImpl.PATH_MAX=512 -Djenkins.branch.WorkspaceLocatorImpl.MAX_LENGTH=512"
```

```sh
systemctl restart jenkins.service
```

[https://plugins.jenkins.io/ldap/](https://plugins.jenkins.io/ldap/)

[https://www.jenkins.io/doc/book/using/using-agents/](https://www.jenkins.io/doc/book/using/using-agents/)

[https://www.jenkins.io/doc/book/managing/system-configuration/](https://www.jenkins.io/doc/book/managing/system-configuration/)

[https://www.jenkins.io/doc/book/security/access-control/](https://www.jenkins.io/doc/book/security/access-control/)

[https://www.jenkins.io/doc/book/security/managing-security/](https://www.jenkins.io/doc/book/security/managing-security/)

[https://www.jenkins.io/doc/book/security/controller-isolation/](https://www.jenkins.io/doc/book/security/controller-isolation/)

[https://www.jenkins.io/doc/book/system-administration/viewing-logs/](https://www.jenkins.io/doc/book/system-administration/viewing-logs/)
