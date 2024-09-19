+++ 
draft = false
date = 2024-09-19T19:01:53+07:00
title = "Internal lab setup notes - Self-hosting an object storage service (S3)"
+++

Use [incus]({{< ref "incus-rhel-profile" >}}) to provision a container to host object storage service:

```sh
# Use managed bridge network for demo/prototype on a Linux workstation
IPV4_ADDR=$(incus network get incusbr0 ipv4.address)
NET_MASK=$(echo ${IPV4_ADDR} | awk -F/ '{print $2}')
SUB_NET=$(echo ${IPV4_ADDR} | awk -F/ '{print $1}' | awk -F\. '{print $1"."$2"."$3}')

incus create images:rockylinux/9/cloud minio --profile rhel

# set static IP for the host
cat <<EOF | incus config set minio cloud-init.network-config -
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
        - ${SUB_NET}.9/${NET_MASK}
      nameservers:
        addresses:
          - ${SUB_NET}.2
EOF

incus start minio
# wait for cloud-init to be done
incus exec minio -- cloud-init status --wait
# Check cloud-init logs in case any failure
incus exec minio -- tail -f /var/log/cloud-init.log
incus exec minio -- tail -f /var/log/cloud-init-output.log
incus restart minio
incus shell minio
```

DNS hostname: `minio.lab.internal`.

```sh
certbot certonly -n --standalone --agree-tos --email postmaster@lab.internal -d minio.lab.internal --server https://ca.lab.internal/acme/acme/directory
systemctl enable --now certbot-renew.timer
```

[https://min.io/docs/minio/linux/operations/install-deploy-manage/deploy-minio-single-node-single-drive.html#minio-snsd](https://min.io/docs/minio/linux/operations/install-deploy-manage/deploy-minio-single-node-single-drive.html#minio-snsd)

[https://min.io/docs/minio/linux/operations/network-encryption.html](https://min.io/docs/minio/linux/operations/network-encryption.html)

`/etc/default/minio`
```sh
# MINIO_ROOT_USER and MINIO_ROOT_PASSWORD sets the root account for the MinIO server.
# This user has unrestricted permissions to perform S3 and administrative API operations on any resource in the deployment.
# Omit to use the default values 'minioadmin:minioadmin'.
# MinIO recommends setting non-default values as a best practice, regardless of environment

MINIO_ROOT_USER=admin
MINIO_ROOT_PASSWORD="S3cret"

# MINIO_VOLUMES sets the storage volume or path to use for the MinIO server.

MINIO_VOLUMES="/mnt/data"

# MINIO_OPTS sets any additional commandline options to pass to the MinIO server.
# For example, `--console-address :9001` sets the MinIO Console listen port
MINIO_OPTS="--console-address :9001 --certs-dir /etc/minio/certs"
```

```sh
mkdir -pv /mnt/data
groupadd -r minio-user
mkdir -pv /etc/minio/certs/CAs
cp /etc/pki/ca-trust/source/anchors/ca.crt /etc/minio/certs/CAs/

useradd -M -r -g minio-user minio-user
chown minio-user:minio-user /mnt/data
chmod 0700 /mnt/data
chown minio-user:minio-user -R /etc/minio
chmod 0700 /mnt/minio
```

`/etc/letsencrypt/renewal-hooks/deploy/main.sh`
```sh
#!/usr/bin/env bash

set -e

MINIO_CERT_DIR=/etc/minio/certs
cp -v ${RENEWED_LINEAGE}/fullchain.pem ${MINIO_CERT_DIR}/public.crt
cp -v ${RENEWED_LINEAGE}/privkey.pem ${MINIO_CERT_DIR}/private.key
chown -v minio-user:minio-user -R ${MINIO_CERT_DIR}
systemctl restart minio
```

```sh
chmod +x /etc/letsencrypt/renewal-hooks/deploy/main.sh
certbot renew --force-renewal
```

```sh
firewall-cmd --permanent --add-port=9000/tcp
firewall-cmd --permanent --add-port=9001/tcp
firewall-cmd --reload
```
