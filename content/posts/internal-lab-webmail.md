+++ 
draft = false
date = 2024-09-17T18:57:36+07:00
title = "Internal lab setup notes - Self-hosting a webmail client"
+++

Use [incus]({{< ref "incus-rhel-profile" >}}) to provision a container to host webmail service:
```sh
# Use managed bridge network for demo/prototype on a Linux workstation
IPV4_ADDR=$(incus network get incusbr0 ipv4.address)
NET_MASK=$(echo ${IPV4_ADDR} | awk -F/ '{print $2}')
SUB_NET=$(echo ${IPV4_ADDR} | awk -F/ '{print $1}' | awk -F\. '{print $1"."$2"."$3}')

incus create images:rockylinux/9/cloud webmail --profile rhel

# set static IP for the host
cat <<EOF | incus config set webmail cloud-init.network-config -
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
        - ${SUB_NET}.7/${NET_MASK}
      nameservers:
        addresses:
          - ${SUB_NET}.2
EOF

incus start webmail
# wait for cloud-init to be done
incus exec webmail -- cloud-init status --wait
# Check cloud-init logs in case any failure
incus exec webmail -- tail -f /var/log/cloud-init.log
incus exec webmail -- tail -f /var/log/cloud-init-output.log
incus restart webmail
incus shell webmail
```

DNS hostname: `webmail.lab.internal`.

```sh
dnf install httpd
systemctl enable --now httpd
```

`/etc/httpd/conf/httpd.conf`:
```
<VirtualHost *:80>
    DocumentRoot "/var/www/html/"
    ServerName webmail.lab.internal
    CustomLog /var/log/httpd/access.log combined
    ErrorLog /var/log/httpd/error.log
</VirtualHost>
```

```sh
certbot certonly -n --apache --agree-tos --email postmaster@lab.internal -d webmail.lab.internal --server https://ca.lab.internal/acme/acme/directory
systemctl enable --now certbot-renew.timer
systemctl restart httpd
```

```sh
firewall-cmd --add-service=https --permanent
firewall-cmd --reload
```

[https://roundcube.net/](https://roundcube.net/)
```sh
wget https://github.com/roundcube/roundcubemail/releases/download/1.6.9/roundcubemail-1.6.9-complete.tar.gz
tar Czxvf /var/www/ roundcubemail-1.6.9-complete.tar.gz
cd /var/www/
mv roundcubemail-1.6.9 roundcubemail
chown -R apache:apache roundcubemail/
```

```sh
dnf module install php:8.2
dnf install php-pcre php-dom php-json php-session php-sockets php-openssl php-mbstring php-filter php-ctype php-intl
dnf install php-pgsql
dnf install php-iconv php-zip php-fileinfo php-exif
dnf install php-gd php-xmlwriter php-ldap
```

`/etc/httpd/conf.d/ssl.conf`:
```
<VirtualHost _default_:443>
DocumentRoot "/var/www/roundcubemail"
ServerName webmail.lab.internal
```

Open [https://webmail.lab.internal/installer](https://webmail.lab.internal/installer).

`/var/www/roundcubemail/config/config.inc.php`:
```php
$config['db_dsnw'] = 'pgsql://user:pass@db.lab.internal/roundcubemail?sslmode=verify-full&sslrootcert=/etc/pki/ca-trust/source/anchors/ca.crt';
$config['imap_host'] = 'ssl://mail.lab.internal';
$config['smtp_host'] = 'ssl://mail.lab.internal';
$config['username_domain'] = 'lab.internal';
$config['identities_level'] = 4;
$config['plugins'] = [
        'additional_message_headers',
        'archive',
        'database_attachments',
        'emoticons',
        'enigma',
        'hide_blockquote',
        'identicon',
        'managesieve',
        'markasjunk',
        'newmail_notifier',
        'show_additional_headers',
        'subscriptions_option',
        'userinfo',
        'zipdownload',
        'new_user_identity',
];
$config['support_url'] = '';
$config['language'] = 'en_US';
$config['htmleditor'] = 1;

$config['imap_debug'] = true;
$config['ldap_debug'] = true;
$config['smtp_debug'] = true;
$config['session_debug'] = true;
$config['disabled_actions'] = ['addressbook'];
$config['address_book_type'] = '';
$config['ldap_public']['lab'] = [
        'name' => 'Internal Lab',
        'hosts' => ['ldaps://idm.lab.internal'],
        'ldap_version' => 3,
        'network_timeout' => 10,
        'user_specific' => false,
        'base_dn' => 'cn=users,cn=accounts,dc=lab,dc=internal',
        'bind_dn' => 'cn=Directory Manager',
        'bind_pass' => 'S3cret',
        'hidden' => true,
        'searchonly' => true,
        'writable' => false,
        'fieldmap' => [
                'name' => 'cn',
                'surname' => 'sn',
                'firstname' => 'givenName',
                'email' => 'mail',
                'photo' => 'jpegPhoto',
        ],
        'sort' => 'cn',
        'scope' => 'sub',
        'filter' => '(objectClass=inetOrgPerson)',
        'config_root_dn' => 'cn=config',
];
$config['autocomplete_addressbooks'] = ['lab'];
$config['collected_recipients'] = false;
$config['collected_senders'] = false;
$config['default_font_size'] = '12pt';
$config['message_show_email'] = true;
```

`/var/www/roundcubemail/plugins/new_user_identity/config.inc.php`:
```php
$config['new_user_identity_addressbook'] = 'lab';
$config['new_user_identity_match'] = 'email';
$config['new_user_identity_onlogin'] = true;
```

`/var/www/roundcubemail/plugins/enigma/config.inc.php`:
```php
$config['enigma_debug'] = true;
$config['enigma_pgp_homedir'] = '/data/enigma';
```

```sh
mkdir -pv /data/enigma
chown apache:apache /data/enigma
chmod 0700 /data/enigma
```

`/var/www/roundcubemail/plugins/managesieve/config.inc.php`:
```php
$config['managesieve_host'] = 'ssl://mail.lab.internal';
```

`/var/www/roundcubemail/plugins/newmail_notifier/config.inc.php`:
```php
$config['newmail_notifier_basic'] = true;
```


