# Upgrading VA to version 2.17.0

Between versions 2.16.2 and 2.17.0 CZERTAINLY, s.r.o was bought by ISS and the product was renamed to ILM (Identity Lifecycle Management). The platform is now developed by [OmniTrust](https://omnitrust.com/).

Release 2.17.0 of ILM Appliance is reflecting this change by renaming most of the files and users from `czertainly` to `ilm`.

## Recomended upgrade path

### 1. Export all your data from VA 2.16.2 by exporting database using

Follow [official documentation](https://docs.otilm.com/docs/certificate-key/installation-guide/deployment/deployment-appliance/operations#backup).

Rewrite `czertainlyuser` to `ilmuser`:
```bash
sed -i "s/czertainlyuser/ilmuser/g"  czertainlydb-<timestamp>.dump.sql
```

### 2. Install new VA 2.17.0 - it comes with new Debian base image.

After the OS update and before ILM installation, copy `/etc/czertainly-ansible/vars/keycloak.yml` from the old appliance.
Copy it to `/etc/ilm-ansible/vars/keycloak.yml` on the new appliance.

### 3. Import your data to VA 2.17.0

Follow [official documentation](https://docs.otilm.com/docs/certificate-key/installation-guide/deployment/deployment-appliance/operations#restore).

## Upgrading without data migration

If you need to upgrade an existing appliance in place, follow the steps below. This procedure was tested for upgrades from 2.16.2 to 2.17.0.

The upgrade changes the product name from CZERTAINLY Appliance to ILM Appliance. It also renames the user and group from `czertainly` to `ilm`, and renames files and directories with the `czertainly-` prefix to `ilm-`. The hostname remains `czertainly`.

### 1. Make backup of complete VA

To be able return in case of failure.

### 2. Save content of `/etc/czertainly-ansible/vars/keycloak.yml` file

This file servers as storage of KeyCloak secret which serves for securing connection between KeyCloak and `czertainly-core`.

### 3. Uninstall old version

Enter TUI, select "Advanced menu", then "Remove RKE2 & CZERTAINLY".

This step will preserve your data inside PostgreSQL database, but it will wipe RKE2 cluster.

### 4. Make root user accessible

Assign SSH key for the `root` user (add your complete SSH key):

```bash
czertainly@czertainly:~$ sudo -i
root@czertainly:~# mkdir -p /root/.ssh
root@czertainly:~# echo "ssh-ed25519 AAAAC3...semik@yggpod" > /root/.ssh/authorized_keys
root@czertainly:~# ^D
logout
czertainly@czertainly:~$ ^D
logout
```
Exit from TUI to release it's SSH connection to make it possible to rename.

### 5. Connect as root and install ilm-appliance-tools

SSH to VA as `root` user and first disable linger service - it is not needed and will cause issues with renaming user and group:

```bash
root@czertainly:~# loginctl disable-linger czertainly
root@czertainly:~# loginctl terminate-user czertainly
```

Then install `ilm-appliance-tools` package:

```bash
root@czertainly:~# apt update
...
root@czertainly:~# apt install ilm-appliance-tools
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following packages will be REMOVED:
  czertainly-appliance-tools
The following NEW packages will be installed:
  ilm-appliance-tools
0 upgraded, 1 newly installed, 1 to remove and 0 not upgraded.
Need to get 0 B/62.2 kB of archives.
After this operation, 2,281 kB disk space will be freed.
Do you want to continue? [Y/n] Y
Get:1 /root/ilm-appliance-tools_2.17.0~rc7_all.deb ilm-appliance-tools all 2.17.0~rc7 [62.2 kB]
(Reading database ... 61618 files and directories currently installed.)
Removing czertainly-appliance-tools (2.16.0) ...
Selecting previously unselected package ilm-appliance-tools.
(Reading database ... 61610 files and directories currently installed.)
Preparing to unpack .../ilm-appliance-tools_2.17.0~rc7_all.deb ...
Unpacking ilm-appliance-tools (2.17.0~rc7) ...
Setting up ilm-appliance-tools (2.17.0~rc7) ...
Renaming OS user czertainly -> ilm
root@czertainly:~# export ANSIBLE_CONFIG=/etc/ilm-ansible/ansible.cfg
root@czertainly:~# ansible-playbook /etc/ilm-ansible/playbooks/ilm-branding.yml

PLAY [Read appliance version] **************************************************************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************************************************
ok: [localhost]

TASK [Read appliance version] **************************************************************************************************************************************
ok: [localhost]

PLAY [Install and configure CZERTAINLY virtual appliance] **********************************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************************************************
ok: [localhost]

TASK [branding : Copy 3KeyCompany templated files] *****************************************************************************************************************
changed: [localhost] => (item={'src': 'motd.j2', 'dest': '/etc/motd'})
changed: [localhost] => (item={'src': 'issue.j2', 'dest': '/etc/issue'})
changed: [localhost] => (item={'src': 'issue.net.j2', 'dest': '/etc/issue.net.j2'})

TASK [branding : copy ILM logo for plymouth] ***********************************************************************************************************************
changed: [localhost] => (item={'src': 'ilm.png', 'dest': '/usr/share/plymouth/themes/spinner/watermark.png'})

TASK [branding : copy ILM logo for grub] ***************************************************************************************************************************
changed: [localhost] => (item={'src': 'ilm.tga', 'dest': '/boot/grub/ilm.tga'})

TASK [branding : Update /etc/defaults/grub with appliance name] ****************************************************************************************************
changed: [localhost]

TASK [branding : Update /etc/defaults/grub with appliance name] ****************************************************************************************************
changed: [localhost]

RUNNING HANDLER [branding : update-grub] ***************************************************************************************************************************
changed: [localhost]

RUNNING HANDLER [branding : update-plymouth] ***********************************************************************************************************************
changed: [localhost]

PLAY RECAP *********************************************************************************************************************************************************
localhost                  : ok=10   changed=7    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### 6. Check `/etc/ilm-ansible/vars/database.yml` file

If you **haven't modified it before**, it was replaced by ilm-appliance-tools package. Your previous system was configured for `czertainlyuser` database user and for `czertainlydb`. Please change it back to:
```yaml
postgres:
  username: czertainlyuser
  password: your-strong-password
  database: czertainlydb
  repository: debian
```
### 7. Check `/etc/ilm-ansible/vars/keycloak.yml` file

That contains value you recorded previously.

### 8. Check `/home/ilm/czertainly-values.custom.yaml` file

If you have previously used custom values files you should have file named `czertainly-values.custom.yaml` in `/home/ilm/` directory. If you have it, rename it to `values.custom.yaml`.

### 9. Install RKE2 & ILM

Install RKE2 & ILM executing `Install` from main menu of TUI. After installation, appliance will be running ILM 2.17.0 with your previous data preserved.