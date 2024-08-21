+++ 
draft = false
date = 2024-08-21T20:06:38+07:00
title = "Internal lab setup notes - Self-hosting a domain name system (DNS)"
+++

Use [incus]({{< ref "incus-rhel-profile" >}}) to provision a container to host DNS service:

```sh
# Use managed bridge network for demo/prototype on a Linux workstation
IPV4_ADDR=$(incus network get incusbr0 ipv4.address)
NET_MASK=$(echo ${IPV4_ADDR} | awk -F/ '{print $2}')
SUB_NET=$(echo ${IPV4_ADDR} | awk -F/ '{print $1}' | awk -F\. '{print $1"."$2"."$3}')

incus create images:rockylinux/9/cloud dns --profile rhel

# set static IP for the host
cat <<EOF | incus config set dns cloud-init.network-config -
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
        - ${SUB_NET}.2/${NET_MASK}
      nameservers:
        addresses:
          - 1.1.1.1
          - 1.0.0.1
EOF

incus start dns
```

[https://technitium.com/dns/](https://technitium.com/dns/).

[https://blog.technitium.com/2022/06/how-to-self-host-your-own-domain-name.html](https://blog.technitium.com/2022/06/how-to-self-host-your-own-domain-name.html).

Zone: `lab.internal`.

[Integrate with resolved](https://linuxcontainers.org/incus/docs/main/howto/network_bridge_resolved/).
