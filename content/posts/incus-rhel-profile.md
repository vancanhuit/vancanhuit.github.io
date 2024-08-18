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

{{< gist vancanhuit b6efa73893dcc6fb19798769c3e28920 >}}


Handling SELinux for `incus-agent` in virtual machine:

[https://github.com/lxc/incus/commit/b2cd793ae4ce016ca7da128cc2d14544c041c801](https://github.com/lxc/incus/commit/b2cd793ae4ce016ca7da128cc2d14544c041c801).

[https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_selinux/changing-selinux-states-and-modes_using-selinux#enabling-selinux-on-systems-that-previously-had-it-disabled_changing-selinux-states-and-modes](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_selinux/changing-selinux-states-and-modes_using-selinux#enabling-selinux-on-systems-that-previously-had-it-disabled_changing-selinux-states-and-modes).

{{< gist vancanhuit d51d2ec51ef07cdd4fc02512a078a4d6 >}}

