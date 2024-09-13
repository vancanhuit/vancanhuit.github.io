+++ 
draft = false
date = 2024-09-13T20:15:22+07:00
title = "Internal lab setup notes - Self-hosting an identity management (IdM) system"
+++

Use [incus]({{< ref "incus-rhel-profile" >}}) to provision a virtual machine to host IdM service:

```sh
# Use managed bridge network for demo/prototype on a Linux workstation
IPV4_ADDR=$(incus network get incusbr0 ipv4.address)
NET_MASK=$(echo ${IPV4_ADDR} | awk -F/ '{print $2}')
SUB_NET=$(echo ${IPV4_ADDR} | awk -F/ '{print $1}' | awk -F\. '{print $1"."$2"."$3}')

incus create images:rockylinux/9/cloud idm --vm --profile rhel --profile vm-config

# set static IP for the host
cat <<EOF | incus config set idm cloud-init.network-config -
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    enp5s0:
      dhcp4: false
      routes:
        - to: 0.0.0.0/0
          via:  ${SUB_NET}.1
      addresses:
        - ${SUB_NET}.4/${NET_MASK}
      nameservers:
        addresses:
          - ${SUB_NET}.2
EOF

incus start idm
# wait for cloud-init to be done
incus exec idm -- cloud-init status --wait
# Check cloud-init logs in case any failure
incus exec idm -- tail -f /var/log/cloud-init.log
incus exec idm -- tail -f /var/log/cloud-init-output.log
incus restart idm
incus shell idm
```

DNS hostname: `idm.lab.internal`.

```sh
# Change SELinux mode to permissive
semanage fcontext -a -t bin_t /var/run/incus_agent/incus-agent
restorecon -R /run/incus_agent
fixfiles -F onboot
# Restart VM
# Change SELinux to enforcing mode
```

```sh
firewall-cmd --permanent --add-service=https
firewall-cmd --permanent --add-service=kerberos
firewall-cmd --permanent --add-service=kpasswd
firewall-cmd --permanent --add-service=ldap
firewall-cmd --permanent --add-service=ldaps
firewall-cmd --reload
```

```sh
dnf install ipa-server
```

[https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/installing_identity_management/assembly_installing-an-ipa-server-without-dns-with-external-ca_installing-identity-management#proc_installing-an-ipa-server-non-interactive-installation-without-dns-with-external-ca_assembly_installing-an-ipa-server-without-dns-with-external-ca](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/installing_identity_management/assembly_installing-an-ipa-server-without-dns-with-external-ca_installing-identity-management#proc_installing-an-ipa-server-non-interactive-installation-without-dns-with-external-ca_assembly_installing-an-ipa-server-without-dns-with-external-ca)

```sh
ipa-server-install --external-ca --realm LAB.INTERNAL --ds-password DM_password --admin-password admin_password --unattended
# Copy /root/ipa.csr file to CA server
```

```sh
# On CA server
step certificate sign ipa.csr /etc/step-ca/certs/root_ca.crt /etc/step-ca/secrets/root_ca_key --password-file /etc/step-ca/password.txt --profile intermediate-ca > ipa.crt
# Copy ipa.crt to idm server
```

```sh
ipa-server-install --external-cert-file=/root/ipa.crt --external-cert-file=/etc/pki/ca-trust/source/anchors/ca.crt --realm LAB.INTERNAL --ds-password DM_password --admin-password admin_password --unattended
```

```
_kerberos-master._tcp.lab.internal. 3600 IN SRV 0 100 88 idm.lab.internal.
_kerberos-master._udp.lab.internal. 3600 IN SRV 0 100 88 idm.lab.internal.
_kerberos._tcp.lab.internal. 3600 IN SRV 0 100 88 idm.lab.internal.
_kerberos._udp.lab.internal. 3600 IN SRV 0 100 88 idm.lab.internal.
_kerberos.lab.internal. 3600 IN TXT "LAB.INTERNAL"
_kerberos.lab.internal. 3600 IN URI 0 100 "krb5srv:m:tcp:idm.lab.internal."
_kerberos.lab.internal. 3600 IN URI 0 100 "krb5srv:m:udp:idm.lab.internal."
_kpasswd._tcp.lab.internal. 3600 IN SRV 0 100 464 idm.lab.internal.
_kpasswd._udp.lab.internal. 3600 IN SRV 0 100 464 idm.lab.internal.
_kpasswd.lab.internal. 3600 IN URI 0 100 "krb5srv:m:tcp:idm.lab.internal."
_kpasswd.lab.internal. 3600 IN URI 0 100 "krb5srv:m:udp:idm.lab.internal."
_ldap._tcp.lab.internal. 3600 IN SRV 0 100 389 idm.lab.internal.
```


```sh
kinit admin
```

```sh
# https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/accessing_identity_management_services/identity-management-security-settings_accessing-idm-services#proc_disabling-anonymous-binds_identity-management-security-settings
dsconf slapd-LAB-INTERNAL config replace nsslapd-allow-anonymous-access=rootdse
# To be compatible with dovecot
dsconf slapd-LAB-INTERNAL pwpolicy set --pwdscheme SSHA512
dsctl slapd-LAB-INTERNAL restart
```

