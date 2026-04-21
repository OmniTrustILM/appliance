# ILM Appliance Builder

Builder of [ILM Appliance](https://docs.czertainly.com/docs/certificate-key/installation-guide/deployment/deployment-appliance/overview), if you are not interested in development you should just [download](https://docs.czertainly.com/docs/certificate-key/installation-guide/deployment/deployment-appliance/overview#download-and-import-image) the published Appliance image. This repository is meant for developers.

## Prerequisites

* a Debian based Linux system with **root access**  - tested on GNU Debian/Linux 11 (Bullseye). Root access is needed for `debootstrap`, mounting qemu disk format, formatting disk image - most of the task is run as root.
* `git` for cloning this repo
* VirtualBox 7.0 (6.0 version doesn't have `--delete-all` option otherwise script should run)
* `qemu-img` and `qemu-nbd` from `qemu-utils`, complete Qemu installation isn't needed
* `debootstrap`
* `dosfstools` for creating FAT partition with EFI stuff

In short:
`apt install git virtualbox qemu-utils debootstrap dosfstools`

## Building

```
git clone https://github.com/OmniTrustILM/appliance.git
cd appliance
sudo ./build-appliance
```
Building requires root permisions as it creates QUEMU virtual disk device.

Finished appliance is exported into a timestamped OVA file under `tmp/` (for example, match it with `tmp/ilm-appliance-*.ova`; the name includes the appliance version, a build timestamp, and may also include `-dev`). The process takes about 7 minutes on i7-6700 CPU @ 3.40GHz.

## Usage of the Appliance

The appliance comes with a preconfigured Debian system. You need to initialize rke2 cluster and install ILM. Please follow the instructions from the [official documentation](https://docs.czertainly.com/docs/certificate-key/installation-guide/deployment/deployment-appliance/initialization).

## Tips for developers

By default Appliance builder uses parameters from [`vars/develop`](./vars/develop) you can make your modifications to that file and pass it as the first argument of the builder, for example:
```
sudo BUILD_PARAMS=vars/local bash ./build-appliance
```
Playbook for ILM installation depends on the following Ansible roles:
  - [ansible-role-ilm-branding](https://github.com/OmniTrustILM/ansible-role-ilm-branding)
  - [ansible-role-http-proxy](https://github.com/OmniTrustILM/ansible-role-http-proxy)
  - [ansible-role-postgres](https://github.com/OmniTrustILM/ansible-role-postgres)
  - [ansible-role-helm](https://github.com/OmniTrustILM/ansible-role-helm)
  - [ansible-role-rke2](https://github.com/OmniTrustILM/ansible-role-rke2)
  - [ansible-role-ilm](https://github.com/OmniTrustILM/ansible-role-ilm)

they are provided by package [`ilm-appliance-tools`](https://github.com/OmniTrustILM/appliance-tools), without any git tracking information. If you need to work on any of them, the best option is to clone a repository of the role you need to work on into the right place under `/etc/ilm-ansible/roles`.

If you want to run Ansible playbooks by hand don't forget to set `ANSIBLE_CONFIG` to the [right](https://github.com/OmniTrustILM/appliance-tools/blob/master/usr/bin/czertainly-tui#L26) values. Typically you can run the installation command from the menu of Text UI.

All Ansible roles have tags. You can run only parts you need to re-run to save your time. For example, when you want just reinstall czeratinly you can do:
```
kubectl delete ns ilm
ANSIBLE_CONFIG=/etc/ilm-ansible/ansible.cfg ansible-playbook /etc/ilm-ansible/playbooks/ilm.yml --tags ilm --skip-tags ilm_sleep10
```
## Tested compatibility of resulting OVA

* VirtualBox 6.1
* VirtualBox 7.0.4 / working environment
* VMPlayer 16.2.4
* VMPlayer 17.0.0

## Notes

Originally was the appliance builder based on `preseed.cfg` file which official way for customizing Debian installation. It is [documented](https://www.debian.org/releases/stable/amd64/apbs02.en.html), but can be sometimes quite tricky to get it working correctly. The main problem with this approach was that it required VT-x instructions, for full virtualization. That is not available in Ubuntu based GitHub runners. With some modifications, it was possible to run it on MacOS based runners, but the building process was taking too long and often was terminated by GitHub after 6 hours. Those modifications for MacOS were replace `genisoimage`=>`mkisofs` and `isohybrid`=>`mkhybrid` which are luckyily dropin replacements.

The actual way of building the appliance is heavily based on the blog post [Building Debian VMs with debootstrap](https://blog.entek.org.uk/technology/2020/06/06/building-debian-vms-with-debootstrap.html). This way of building the appliance is much faster and it runs even on Ubuntu runners on GitHub.
