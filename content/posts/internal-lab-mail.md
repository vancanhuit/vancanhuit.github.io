+++ 
draft = false
date = 2024-09-15T18:07:29+07:00
title = "Internal lab setup notes - Self-hosting a mail service (SMTP + IMAP)"
+++

Use [incus]({{< ref "incus-rhel-profile" >}}) to provision a container to host mail service:

```sh
# Use managed bridge network for demo/prototype on a Linux workstation
IPV4_ADDR=$(incus network get incusbr0 ipv4.address)
NET_MASK=$(echo ${IPV4_ADDR} | awk -F/ '{print $2}')
SUB_NET=$(echo ${IPV4_ADDR} | awk -F/ '{print $1}' | awk -F\. '{print $1"."$2"."$3}')

incus create images:rockylinux/9/cloud mail --profile rhel

# set static IP for the host
cat <<EOF | incus config set mail cloud-init.network-config -
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
        - ${SUB_NET}.6/${NET_MASK}
      nameservers:
        addresses:
          - ${SUB_NET}.2
EOF

incus start mail
# wait for cloud-init to be done
incus exec mail -- cloud-init status --wait
# Check cloud-init logs in case any failure
incus exec mail -- tail -f /var/log/cloud-init.log
incus exec mail -- tail -f /var/log/cloud-init-output.log
incus restart mail
incus shell mail
```

DNS hostname: `mail.lab.internal`.

[https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/deploying_mail_servers](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/deploying_mail_servers)

```sh
certbot certonly -n --standalone --agree-tos --email postmaster@lab.internal -d mail.lab.internal --server https://ca.lab.internal/acme/acme/directory
systemctl enable --now certbot-renew.timer
```

```sh
firewall-cmd --permanent --add-service={imaps,smtps}
firewall-cmd --reload
```

## Dovecot

`/etc/dovecot/conf.d/10-ssl.conf`
```
ssl = required
ssl_cert = </etc/letsencrypt/live/mail.lab.internal/fullchain.pem
ssl_key = </etc/letsencrypt/live/mail.lab.internal/privkey.pem
```

`/etc/dovecot/dovecot.conf`
```
protocols = imap lmtp
verbose_proctitle = yes
```

`/etc/dovecot/conf.d/10-logging.conf`
```
auth_verbose = yes
auth_debug = yes
mail_debug = yes
verbose_ssl = yes
```

`/etc/dovecot/conf.d/10-mail.conf`
```conf
mail_location = maildir:~/Maildir
namespace inbox {
  #type = private
  separator = /
  prefix = INBOX/
  #location =
  inbox = yes
  #hidden = no
  #list = yes
  #subscriptions = yes
  # See 15-mailboxes.conf for definitions of special mailboxes.
}
```

`/etc/dovecot/conf.d/15-mailboxes.conf`
```
namespace inbox {
  mailbox Drafts {
    special_use = \Drafts
    auto = subscribe
  }
  mailbox Junk {
    special_use = \Junk
    auto = subscribe
  }
  mailbox Trash {
    special_use = \Trash
    auto = subscribe
  }
  mailbox Sent {
    special_use = \Sent
    auto = subscribe
  }
  mailbox Archive/ {
    special_use = \Archive
    auto = subscribe
  }
}
```

`/etc/dovecot/conf.d/10-auth.conf`
```
disable_plaintext_auth = yes
auth_username_format = %Lu
auth_mechanisms = plain login
!include auth-ldap.conf.ext
```

`/etc/dovecot/conf.d/auth-ldap.conf.ext`
```
passdb {
  driver = ldap
  args = /etc/dovecot/dovecot-ldap.conf.ext
}
userdb {
  driver = prefetch
}
userdb {
  driver = ldap
  args = /etc/dovecot/dovecot-ldap.conf.ext
}
```

`/etc/dovecot/dovecot-ldap.conf.ext`
```
uris = ldaps://idm.lab.internal
dn = "cn=Directory Manager"
dnpass = "S3cret"
tls_require_cert = hard
auth_bind = no
base = cn=users,cn=accounts,dc=lab,dc=internal
user_attrs = =home=/data/vmail/%Ld/%Ln,=uid=10000,=gid=10000
user_filter = (&(objectClass=inetOrgPerson)(mail=%u))
pass_attrs = userPassword=password,\
  =userdb_home=/data/vmail/%Ld/%Ln,=userdb_uid=10000,=userdb_gid=10000
pass_filter = (&(objectClass=inetOrgPerson)(mail=%u))
iterate_attrs = mail=user
iterate_filter = (objectClass=inetOrgPerson)
```

```sh
groupadd -g 10000 vmail
useradd -u 10000 -g 10000 -s /sbin/nologin vmail
mkdir -pv /data/vmail
chown vmail:vmail /data/vmail
chmod 0700 /data/vmail
```

`/etc/dovecot/conf.d/10-master.conf`
```
service lmtp {
  inet_listener lmtp {
    address = 127.0.0.1
    port = 24
  }
}
service auth {
  unix_listener /var/spool/postfix/private/auth {
    mode = 0666
    user = postfix
    group = postfix
  }
}
```

`/etc/dovecot/conf.d/20-managesieve.conf`
```
protocols = $protocols sieve
service managesieve-login {
  inet_listener sieve {
    ssl = yes
    port = 4190
  }
}
```

```sh
systemctl enable --now dovecot
```

## Postfix


`/etc/postfix/main.cf`
```
alias_database = hash:/etc/aliases
alias_maps = hash:/etc/aliases
broken_sasl_auth_clients = yes
command_directory = /usr/sbin
compatibility_level = 2
daemon_directory = /usr/libexec/postfix
data_directory = /var/lib/postfix
debug_peer_level = 2
debugger_command = PATH=/bin:/usr/bin:/usr/local/bin:/usr/X11R6/bin ddd $daemon_directory/$process_name $process_id & sleep 5
html_directory = no
inet_interfaces = all
inet_protocols = all
mail_owner = postfix
mailq_path = /usr/bin/mailq.postfix
manpage_directory = /usr/share/man
meta_directory = /etc/postfix
mydestination = $myhostname, localhost.$mydomain, localhost
mydomain = lab.internal
myhostname = mail.lab.internal
myorigin = $mydomain
newaliases_path = /usr/bin/newaliases.postfix
queue_directory = /var/spool/postfix
readme_directory = /usr/share/doc/postfix/README_FILES
relay_domains = $mydomain
sample_directory = /usr/share/doc/postfix/samples
sendmail_path = /usr/sbin/sendmail.postfix
setgid_group = postdrop
shlib_directory = /usr/lib64/postfix
smtp_tls_CAfile = /etc/pki/tls/certs/ca-bundle.crt
smtp_tls_CApath = /etc/pki/tls/certs
smtp_tls_security_level = may
smtpd_recipient_restrictions = reject_unverified_recipient
smtpd_sasl_auth_enable = yes
smtpd_sasl_path = private/auth
smtpd_sasl_security_options = noanonymous, noplaintext
smtpd_sasl_tls_security_options = noanonymous
smtpd_sasl_type = dovecot
smtpd_tls_auth_only = yes
smtpd_tls_cert_file = /etc/letsencrypt/live/mail.lab.internal/fullchain.pem
smtpd_tls_key_file = /etc/letsencrypt/live/mail.lab.internal/privkey.pem
smtpd_tls_security_level = may
transport_maps = hash:/etc/postfix/transport
unknown_local_recipient_reject_code = 550
unverified_recipient_reject_code = 577
```

`/etc/postifx/transport`
```
lab.internal    lmtp:[127.0.0.1]
```

```sh
postmap /etc/postfix/transport
```

`/etc/postfix/master.cf`
```
smtps     inet  n       -       n       -       -       smtpd
  -o syslog_name=postfix/smtps
  -o smtpd_tls_wrappermode=yes
```

```sh
systemctl enable --now postfix
```
