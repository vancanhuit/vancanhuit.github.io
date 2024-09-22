+++ 
draft = false
date = 2024-09-20T18:55:53+07:00
title = "Internal lab setup notes - Self-hosting a Git hosting service"
+++

Use [incus]({{< ref "incus-rhel-profile" >}}) to provision a container to host [Gitea](https://about.gitea.com/) service:

```sh
# Use managed bridge network for demo/prototype on a Linux workstation
IPV4_ADDR=$(incus network get incusbr0 ipv4.address)
NET_MASK=$(echo ${IPV4_ADDR} | awk -F/ '{print $2}')
SUB_NET=$(echo ${IPV4_ADDR} | awk -F/ '{print $1}' | awk -F\. '{print $1"."$2"."$3}')

incus create images:rockylinux/9/cloud gitea --profile rhel

# set static IP for the host
cat <<EOF | incus config set gitea cloud-init.network-config -
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
        - ${SUB_NET}.10/${NET_MASK}
      nameservers:
        addresses:
          - ${SUB_NET}.2
EOF

incus start gitea
# wait for cloud-init to be done
incus exec gitea -- cloud-init status --wait
# Check cloud-init logs in case any failure
incus exec gitea -- tail -f /var/log/cloud-init.log
incus exec gitea -- tail -f /var/log/cloud-init-output.log
incus restart gitea
incus shell gitea
```

DNS hostname: `gitea.lab.internal`.

[https://docs.gitea.com/installation/install-from-binary](https://docs.gitea.com/installation/install-from-binary)

[https://docs.gitea.com/administration/https-setup#using-acme-default-lets-encrypt](https://docs.gitea.com/administration/https-setup#using-acme-default-lets-encrypt)

```sh
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

```ini
APP_NAME = Gitea: Git with a cup of tea
RUN_USER = git
WORK_PATH = /var/lib/gitea
RUN_MODE = prod

[database]
DB_TYPE = postgres
HOST = db.lab.internal:5432
NAME = gitea
USER = svc_db
PASSWD = `S3cret`
SCHEMA =
SSL_MODE = verify-full
PATH = /var/lib/gitea/data/gitea.db
LOG_SQL = false

[repository]
ROOT = /var/lib/gitea/data/gitea-repositories

[server]
SSH_DOMAIN = gitea.lab.internal
DOMAIN = gitea.lab.internal
PROTOCOL = https
HTTP_PORT = 443
ROOT_URL = https://gitea.lab.internal/
APP_DATA_PATH = /var/lib/gitea/data
DISABLE_SSH = false
SSH_PORT = 22
LFS_START_SERVER = true
LFS_JWT_SECRET = Zy4BN0sFLqTpZjwRko7P5W_xfhsAeNcZX59so6qu378
OFFLINE_MODE = false
ENABLE_ACME = true
ACME_ACCEPTTOS = true
ACME_URL = https://ca.lab.internal/acme/acme/directory
ACME_DIRECTORY = https
ACME_EMAIL = postmaster@lab.internal

[lfs]
PATH = /var/lib/gitea/data/lfs

[mailer]
ENABLED = true
SMTP_ADDR = mail.lab.internal
SMTP_PORT = 465
FROM = Gitea <gitea-no-reply@lab.internal>
USER =
PASSWD =

[service]
REGISTER_EMAIL_CONFIRM = false
ENABLE_NOTIFY_MAIL = true
DISABLE_REGISTRATION = true
ALLOW_ONLY_EXTERNAL_REGISTRATION = false
ENABLE_CAPTCHA = false
REQUIRE_SIGNIN_VIEW = true
DEFAULT_KEEP_EMAIL_PRIVATE = false
DEFAULT_ALLOW_CREATE_ORGANIZATION = true
DEFAULT_ENABLE_TIMETRACKING = true
NO_REPLY_ADDRESS = noreply.lab.internal

[service.explore]
REQUIRE_SIGNIN_VIEW = true
DISABLE_USERS_PAGE = true

[openid]
ENABLE_OPENID_SIGNIN = false
ENABLE_OPENID_SIGNUP = false

[cron.update_checker]
ENABLED = true

[session]
PROVIDER = file

[log]
MODE = console
LEVEL = info
ROOT_PATH = /var/lib/gitea/log

[repository.pull-request]
DEFAULT_MERGE_STYLE = squash

[repository.signing]
DEFAULT_TRUST_MODEL = committer

[security]
INSTALL_LOCK = true
INTERNAL_TOKEN = S3cret
PASSWORD_HASH_ALGO = pbkdf2

[oauth2]
JWT_SECRET = S3cret

[storage]
STORAGE_TYPE = minio
MINIO_ENDPOINT = minio.lab.internal:9000
MINIO_ACCESS_KEY_ID = S3cret
MINIO_SECRET_ACCESS_KEY = S3cret
MINIO_BUCKET = gitea
MINIO_LOCATION = vn-hcm
MINIO_USE_SSL = true
MINIO_INSECURE_SKIP_VERIFY = false
SERVE_DIRECT = true

[webhook]
ALLOWED_HOST_LIST = private

[actions]
ENABLED = false

[ui]
DEFAULT_SHOW_FULL_NAME = true
```

[https://docs.gitea.com/usage/authentication](https://docs.gitea.com/usage/authentication)

[https://docs.gitea.com/administration/config-cheat-sheet](https://docs.gitea.com/administration/config-cheat-sheet)
