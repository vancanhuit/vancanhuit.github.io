+++
title = 'Incus notes'
date = 2024-08-16T21:39:16+07:00
draft = false
+++

[https://linuxcontainers.org/incus/docs/main/](https://linuxcontainers.org/incus/docs/main/)

[https://linuxcontainers.org/incus/docs/main/profiles/](https://linuxcontainers.org/incus/docs/main/profiles/).

[https://linuxcontainers.org/incus/docs/main/cloud-init/](https://linuxcontainers.org/incus/docs/main/cloud-init/).

[https://cloudinit.readthedocs.io/en/latest/index.html](https://cloudinit.readthedocs.io/en/latest/index.html).

[https://linuxcontainers.org/incus/docs/main/howto/instances_configure/](https://linuxcontainers.org/incus/docs/main/howto/instances_configure/).

[https://linuxcontainers.org/incus/docs/main/reference/devices_disk/](https://linuxcontainers.org/incus/docs/main/reference/devices_disk/).

[https://linuxcontainers.org/incus/docs/main/reference/devices_nic/](https://linuxcontainers.org/incus/docs/main/reference/devices_nic/).

[https://linuxcontainers.org/incus/docs/main/howto/network_bridge_firewalld/](https://linuxcontainers.org/incus/docs/main/howto/network_bridge_firewalld/).

[https://linuxcontainers.org/incus/docs/main/howto/network_bridge_resolved/](https://linuxcontainers.org/incus/docs/main/howto/network_bridge_resolved/).

[https://linuxcontainers.org/incus/docs/main/explanation/storage/](https://linuxcontainers.org/incus/docs/main/explanation/storage/).

[https://docs.rockylinux.org/books/lxd_server/06-profiles/#rocky-linux-macvlan](https://docs.rockylinux.org/books/lxd_server/06-profiles/#rocky-linux-macvlan).

[https://linuxcontainers.org/incus/docs/main/explanation/clustering/](https://linuxcontainers.org/incus/docs/main/explanation/clustering/).

Profile for Red Hat based distributions:

```yaml
config:
  cloud-init.vendor-data: |
    ## template: jinja
    #cloud-config
    hostname: "{{ ds.meta_data.instance_id }}.lab.internal"
    package_upgrade: true
    yum_repos:
      epel-release:
        name: Extra Packages for Enterprise Linux $releasever - $basearch
        baseurl: https://dl.fedoraproject.org/pub/epel/$releasever/Everything/$basearch/
        metalink: https://mirrors.fedoraproject.org/metalink?repo=epel-$releasever&arch=$basearch&infra=$infra&content=$contentdir
        countme: 1
        gpgcheck: true
        gpgkey: https://dl.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-$releasever
    packages:
      - bash-completion
      - openssh
      - curl
      - wget
      - htop
      - vim
      - tar
      - man
      - firewalld
      - certbot
    timezone: Asia/Ho_Chi_Minh
    runcmd:
      - systemctl enable --now firewalld.service
      - firewall-cmd --remove-service=cockpit --permanent
      - firewall-cmd --remove-service=dhcpv6-client --permanent
      - firewall-cmd --add-service=http --permanent
      - firewall-cmd --reload
      - mandb
  limits.cpu: "1"
  limits.memory: 1GiB
description: RHEL-based distro Incus profile
devices:
  eth0:
    name: eth0
    network: incusbr0
    type: nic
  root:
    path: /
    pool: default
    type: disk
name: rhel
used_by: []
project: default
```

```yaml
config:
  limits.memory: 2GiB
description: "Profile for VM"
devices:
  agent:
    source: agent:config
    type: disk
name: vm-config
used_by: []
project: default
```

```sh 
incus launch images:rockylinux/9/cloud test-01 --profile rhel
incus launch images:rockylinux/9/cloud test-02 --vm --profile rhel --profile vm-config
```

Handling SELinux for `incus-agent` in virtual machine:

[https://github.com/lxc/incus/commit/b2cd793ae4ce016ca7da128cc2d14544c041c801](https://github.com/lxc/incus/commit/b2cd793ae4ce016ca7da128cc2d14544c041c801).

[https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_selinux/changing-selinux-states-and-modes_using-selinux#enabling-selinux-on-systems-that-previously-had-it-disabled_changing-selinux-states-and-modes](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_selinux/changing-selinux-states-and-modes_using-selinux#enabling-selinux-on-systems-that-previously-had-it-disabled_changing-selinux-states-and-modes).

```sh
semanage fcontext -a -t bin_t /var/run/incus_agent/incus-agent
restorecon -R /run/incus-agent
fixfiles -F onboot
```
