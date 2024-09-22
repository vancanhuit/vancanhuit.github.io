+++ 
draft = false
date = 2024-09-18T19:16:55+07:00
title = "Internal lab setup notes - Self-hosting an Identity Provider (IdP)"
+++

Use [incus]({{< ref "incus-rhel-profile" >}}) to provision a container to host IdP service:
```sh
# Use managed bridge network for demo/prototype on a Linux workstation
IPV4_ADDR=$(incus network get incusbr0 ipv4.address)
NET_MASK=$(echo ${IPV4_ADDR} | awk -F/ '{print $2}')
SUB_NET=$(echo ${IPV4_ADDR} | awk -F/ '{print $1}' | awk -F\. '{print $1"."$2"."$3}')

incus create images:rockylinux/9/cloud keycloak --profile rhel

# set static IP for the host
cat <<EOF | incus config set keycloak cloud-init.network-config -
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
        - ${SUB_NET}.8/${NET_MASK}
      nameservers:
        addresses:
          - ${SUB_NET}.2
EOF

incus start keycloak
# wait for cloud-init to be done
incus exec keycloak -- cloud-init status --wait
# Check cloud-init logs in case any failure
incus exec keycloak -- tail -f /var/log/cloud-init.log
incus exec keycloak -- tail -f /var/log/cloud-init-output.log
incus restart keycloak
incus shell keycloak
```

DNS hostname: `keycloak.lab.internal`.

```sh
certbot certonly -n --standalone --agree-tos --email postmaster@lab.internal -d keycloak.lab.internal --server https://ca.lab.internal/acme/acme/directory
systemctl enable --now certbot-renew.timer
```

```sh
firewall-cmd --add-service=https --permanent
firewall-cmd --reload
```

[https://www.keycloak.org/](https://www.keycloak.org/)

[https://www.keycloak.org/server/configuration](https://www.keycloak.org/server/configuration)

[https://www.keycloak.org/server/configuration-production](https://www.keycloak.org/server/configuration-production)

```sh
dnf install java-21-openjdk-headless
```

```sh
wget https://github.com/keycloak/keycloak/releases/download/25.0.5/keycloak-25.0.5.tar.gz
tar Czxvf /opt keycloak-25.0.5.tar.gz
cd /opt
chown -R root:root keycloak-25.0.5/
chmod 0700 /opt/keycloak-25.0.5
ln -sfn keycloak-25.0.5 keycloak
mkdir -pv /etc/keycloak
chmod 0700 /etc/keycloak
cd /etc/keycloak/
cp /opt/keycloak/conf/keycloak.conf .
```

`/etc/keycloak/keycloak.conf`
```
# Basic settings for running in production. Change accordingly before deploying the server.

# Database

# The database vendor.
db=postgres

# The username of the database user.
db-username=svc_db
db-url-host=db.lab.internal
db-url-port=5432
db-url-database=keycloak
db-url-properties=?sslmode=verify-full&sslrootcert=/etc/pki/ca-trust/source/anchors/ca.crt

# The file path to a server certificate or certificate chain in PEM format.
https-certificate-file=/etc/letsencrypt/live/keycloak.lab.internal/fullchain.pem

# The file path to a private key in PEM format.
https-certificate-key-file=/etc/letsencrypt/live/keycloak.lab.internal/privkey.pem
https-port=443

# Hostname for the Keycloak server.
hostname=keycloak.lab.internal

config-keystore=/etc/keycloak/keystore.p12
config-keystore-password=keystorepass
config-keystore-type=PKCS12
```

`/etc/keycloak/keystore.p12`:
```sh
keytool -importpass -alias kc.db-password -keystore keystore.p12 -storepass keystorepass -storetype PKCS12 -v
```

```sh
chmod 0600 /etc/keycloak/keycloak.conf
chmod 0600 /etc/keycloak/keycloak.p12
```

```sh
/opt/keycloak/bin/kc.sh build --db=postgres
```

```sh
export KEYCLOAK_ADMIN=admin
export KEYCLOAK_ADMIN_PASSWORD=S3cret
/opt/keycloak/bin/kc.sh --config-file /etc/keycloak/keycloak.conf start --optimized
```

```sh
unset KEYCLOAK_ADMIN
unset KEYCLOAK_ADMIN_PASSWORD
```

`/etc/systemd/system/keycloak.service`
```
[Unit]
Description=Keycloak IAM
Documentation=https://keycloak.org/
After=network.target
[Service]
Type=simple
ExecStart=/opt/keycloak/bin/kc.sh --config-file /etc/keycloak/keycloak.conf start --optimized
[Install]
WantedBy=multi-user.target
```

```sh
systemctl daemon-reload
systemctl enable --now keycloak
```

[https://www.keycloak.org/docs/latest/server_admin/index.html#_ldap](https://www.keycloak.org/docs/latest/server_admin/index.html#_ldap)

[https://www.keycloak.org/docs/latest/server_admin/index.html#configuring-realms](https://www.keycloak.org/docs/latest/server_admin/index.html#configuring-realms)

[https://www.keycloak.org/docs/latest/server_admin/index.html#con-oidc_server_administration_guide](https://www.keycloak.org/docs/latest/server_admin/index.html#con-oidc_server_administration_guide)

[https://www.keycloak.org/docs/latest/server_admin/index.html#configuring-authentication_server_administration_guide](https://www.keycloak.org/docs/latest/server_admin/index.html#configuring-authentication_server_administration_guide)


