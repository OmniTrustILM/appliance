# Testing the Appliance behind HTTP Proxy

This guide is meant for developers who want to test the Appliance behind an HTTP proxy.

## Prerequisites

VirtualBox with configured Host-only Network (File -> Tools -> Network manager). In this guide I use vboxnet0 with IP network 192.168.56.0/24

### DNS
DNS server running on same host as VirtualBox. DNSMASQ with config `/etc/dnsmasq.conf`:
```conf
interface=vboxnet0           # VirtualBox host-only interface name
listen-address=192.168.56.1  # Host-only IP
bind-interfaces
server=192.168.1.1           # Upstream DNS
```
will do the trick.

### HTTP Proxy
And finally Squid proxy server. I used squid-5.7-2+deb12u2 on Debian Bookworm. With following config `/etc/squid/conf.d/vboxnet0.conf`:

```conf
# Allow access from your host-only network (e.g., 192.168.56.0/24)
acl hostonlynet src 192.168.56.0/24

# Debian package system - needed for system updates
acl debian_packages dstdomain deb.debian.org security.debian.org
# 3Key Resources - debian package + oci helm charts
acl 3key_resources dstdomain deb.czertainly.com harbor.3key.company
# Resources for helm package + helm diff plugin
#   https://github.com/CZERTAINLY/ansible-role-helm/blob/develop/tasks/helm.yml#L28
#   https://github.com/CZERTAINLY/ansible-role-helm/blob/develop/tasks/helm_diff.yml#L13
acl helm_resources dstdomain baltocdn.com github.com release-assets.githubusercontent.com
# Resources for Docker needed by RKE2, cert-manager and CZERTAINLY
acl docker_resources dstdomain index.docker.io auth.docker.io production.cloudflare.docker.com registry-1.docker.io
# Resources for RKE2
acl rke2_resources dstdomain get.rke2.io update.rke2.io
# Resources for local-path-provisioner
#   docker-images-prod.6aa30f8b08e16409b46e0173d6de2f56.r2.cloudflarestorage.com:443 seems like crazy name
#   that is why are using regex.
acl local_path_resources dstdom_regex ^raw\.githubusercontent\.com$ ^docker-images-prod\..*\.cloudflarestorage\.com$
# Resources for cert-manager - images are at cdn01.quay.io I rather use regex.
acl cert_mananager_resources dstdom_regex ^charts\.jetstack\.io$ ^quay\.io$ ^cdn.*\.quay\.io$

# Access rules: Only allow source from hostonlynet to allowed_domains
http_access allow hostonlynet debian_packages
http_access allow hostonlynet 3key_resources
http_access allow hostonlynet helm_resources
http_access allow hostonlynet docker_resources
http_access allow hostonlynet rke2_resources
http_access allow hostonlynet local_path_resources
http_access allow hostonlynet cert_mananager_resources

# Deny all other traffic from hostonlynet
http_access deny hostonlynet

# Usual final deny all
http_access deny all
```

## Appliance testing

Exec appliance, gain root and manually do following tasks.

Configure DNS Server as Host-only adapter doesn't provide any:
```
echo "nameserver 192.168.56.1" > /etc/resolv.conf
```

Add default route (without it rke2 install fails):
```
echo '#!/bin/sh

ip route add default via 192.168.56.1 dev eth0
exit 0
' > /etc/network/if-up.d/eth0-poststart
chmod +x /etc/network/if-up.d/eth0-poststart
```

Configure HTTP proxy using TUI to use https://192.168.56.1:3128/

Reboot to make sure changes take effect.

**Commence testing behind the HTTP proxy.**