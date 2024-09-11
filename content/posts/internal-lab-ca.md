+++ 
draft = false
date = 2024-09-11T20:08:10+07:00
title = "Internal lab setup notes - Self-hosting a Certificate Authority (CA) service"
+++

Use [incus]({{< ref "incus-rhel-profile" >}}) to provision a container to host CA service:

```sh
# Use managed bridge network for demo/prototype on a Linux workstation
IPV4_ADDR=$(incus network get incusbr0 ipv4.address)
NET_MASK=$(echo ${IPV4_ADDR} | awk -F/ '{print $2}')
SUB_NET=$(echo ${IPV4_ADDR} | awk -F/ '{print $1}' | awk -F\. '{print $1"."$2"."$3}')

incus create images:rockylinux/9/cloud ca --profile rhel

# set static IP for the host
cat <<EOF | incus config set ca cloud-init.network-config -
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
        - ${SUB_NET}.3/${NET_MASK}
      nameservers:
        addresses:
          - ${SUB_NET}.2
EOF

incus start ca
# wait for cloud-init to be done
incus exec ca -- cloud-init status --wait
# Check cloud-init logs in case any failure
incus exec ca -- tail -f /var/log/cloud-init.log
incus exec ca -- tail -f /var/log/cloud-init-output.log
incus restart ca
incus shell ca
```

DNS hostname: `ca.lab.internal`.

[https://smallstep.com/docs/step-ca/installation/#redhat](https://smallstep.com/docs/step-ca/installation/#redhat)

[https://smallstep.com/docs/step-ca/getting-started/](https://smallstep.com/docs/step-ca/getting-started/)

[https://smallstep.com/docs/step-ca/basic-certificate-authority-operations/](https://smallstep.com/docs/step-ca/basic-certificate-authority-operations/)

[https://smallstep.com/docs/step-ca/acme-basics/](https://smallstep.com/docs/step-ca/acme-basics/)

[https://smallstep.com/docs/step-ca/configuration/](https://smallstep.com/docs/step-ca/configuration/)

[https://smallstep.com/docs/step-ca/certificate-authority-server-production/#create-a-service-user-to-run-step-ca](https://smallstep.com/docs/step-ca/certificate-authority-server-production/#create-a-service-user-to-run-step-ca)

[https://eff-certbot.readthedocs.io/en/latest/](https://eff-certbot.readthedocs.io/en/latest/)
