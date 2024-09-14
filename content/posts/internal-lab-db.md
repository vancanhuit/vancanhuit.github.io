+++ 
draft = false
date = 2024-09-14T11:25:49+07:00
title = "Internal lab setup notes - Self-hosting a SQL database (DB)"
+++

Use [incus]({{< ref "incus-rhel-profile" >}}) to provision a container to host DB service:

```sh
# Use managed bridge network for demo/prototype on a Linux workstation
IPV4_ADDR=$(incus network get incusbr0 ipv4.address)
NET_MASK=$(echo ${IPV4_ADDR} | awk -F/ '{print $2}')
SUB_NET=$(echo ${IPV4_ADDR} | awk -F/ '{print $1}' | awk -F\. '{print $1"."$2"."$3}')

incus create images:rockylinux/9/cloud db --profile rhel

# set static IP for the host
cat <<EOF | incus config set db cloud-init.network-config -
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
        - ${SUB_NET}.5/${NET_MASK}
      nameservers:
        addresses:
          - ${SUB_NET}.2
EOF

incus start db
# wait for cloud-init to be done
incus exec db -- cloud-init status --wait
# Check cloud-init logs in case any failure
incus exec db -- tail -f /var/log/cloud-init.log
incus exec db -- tail -f /var/log/cloud-init-output.log
incus restart db
incus shell db
```

DNS hostname: `db.lab.internal`.

[https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_and_using_database_servers/using-postgresql_configuring-and-using-database-servers](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_and_using_database_servers/using-postgresql_configuring-and-using-database-servers)

[https://smallstep.com/docs/tutorials/acme-protocol-acme-clients/#certbot](https://smallstep.com/docs/tutorials/acme-protocol-acme-clients/#certbot)

```sh
certbot certonly -n --standalone --agree-tos --email postmaster@lab.internal -d db.lab.internal --server https://ca.lab.internal/acme/acme/directory
```

`/etc/letsencrypt/renewal-hooks/deploy/main.sh`
```sh
#!/usr/bin/env bash

set -e

PG_DATA_DIR=/var/lib/pgsql/data
cp -v ${RENEWED_LINEAGE}/fullchain.pem ${PG_DATA_DIR}/server.crt
cp -v ${RENEWED_LINEAGE}/privkey.pem ${PG_DATA_DIR}/server.key

chown -v postgres:postgres ${PG_DATA_DIR}/server.{key,crt}
chmod -v 0400 ${PG_DATA_DIR}/server.key
```

```sh
chmod +x /etc/letsencrypt/renewal-hooks/deploy/main.sh
```

```sh
certbot renew --force-renewal
```

```sh
systemctl enable --now certbot-renew.timer
```

