# Upgrading VA to version 2.19.0

This document covers the steps needed when upgrading from appliance 2.17.0 to 2.19.0, if you are upgrading from lower version check [previous documentation for upgrading to 2.17.0](upgrading2-2.17.0.md). Between appliance 2.17.0 and 2.19.0 the ILM Helm chart went through **two**
releases that need manual work, 2.18.0 and 2.19.0. Read the official
[Helm upgrade notes](https://docs.otilm.com/docs/certificate-key/installation-guide/deployment/deployment-helm/upgrading)
alongside this document; what follows is the appliance-specific version of it.

Two things bite in this upgrade, and each one can lock you out if it is done in the wrong order:

| what | comes from | symptom if skipped |
|---|---|---|
| the trusted root CA changed name | chart 2.18.0 | administrators cannot log in, the browser fails the TLS handshake |
| the Keycloak realm was renamed CZERTAINLY -> ILM | chart 2.18.0 | Keycloak fails to start, `--import-realm` hits a duplicate primary key |

> **Order matters.** The Keycloak migration runs **before** `helm upgrade`,
> while the old Keycloak is still running.

## 0. Back up first

1. A snapshot of the whole virtual machine, so you can go back.
2. A database dump, see the
   [operations documentation](https://docs.otilm.com/docs/certificate-key/installation-guide/deployment/deployment-appliance/operations#backup).
3. A copy of `/etc/ilm-ansible/vars/` - it holds `keycloak.yml` (the Keycloak
   client secret).
4. A copy of `/home/ilm/` - it holds `values.custom.yaml` and the file with
   trusted certificates from the next step, both are read again by every run of
   the installation. Together with the directory above this is the
   [minimum backup](https://docs.otilm.com/docs/certificate-key/installation-guide/deployment/deployment-appliance/operations#backup)
   of an appliance.

## 1. Keep trusting your current root CA

> This section matter only if you were using pre-provisioned certificates. Feel free to skip it if you were using your own CA.

Chart 2.18.0 replaced the built-in root certificate authority. Old appliances trust [`CN=CZERTAINLY Dummy Root CA`](https://github.com/OmniTrustILM/helm-charts/blob/2.17.0/dummy-certificates/certs/root-ca.cert.pem); 2.18.0 and later ship [`CN=Dummy Root CA`](https://github.com/OmniTrustILM/helm-charts/blob/2.19.0/dummy-certificates/certs/root-ca.cert.pem). An administrator certificate issued by a pre-2.19.0 appliance uses the old root; you can continue to use it as long as both the old and new root CAs are trusted.

To ensure continued trust, you need to combine the [old](https://github.com/OmniTrustILM/helm-charts/blob/2.17.0/dummy-certificates/certs/root-ca.cert.pem) and [new](https://github.com/OmniTrustILM/helm-charts/blob/2.19.0/dummy-certificates/certs/root-ca.cert.pem) root CAs into a single file. Configure both of them as [custom trusted certificate](https://docs.otilm.com/docs/certificate-key/installation-guide/deployment/deployment-appliance/TUI/main-menu#configure-custom-trusted-certificates). Place them for example into file `/home/ilm/trusted-list.pem`.

Keep that file in the home directory of the `ilm` user. It is not copied anywhere, it is read from that path again by every run of the ILM installation, so it has to stay there for as long as the appliance is trusting those certificates.

> The configured file **replaces** the certificates shipped by the chart, it is not added to them. A file with the old root alone would repair the old certificates and break the new ones, which is why both roots have to be in that single file.

You don't need to re-run Install, because in the later steps we are goin to upgrade the appliance using the new RKE2 and Traefik setup. The file is read by the ILM installation task, so configuring it any time before the installation in step 6 is fine.

## 2. Migrate the Keycloak realm (needed by chart 2.18.0)

> Only for appliances that were installed as CZERTAINLY 2.17.0 and older.
The realm is renamed in place, keeping every UUID, user, session and group
membership; the script is idempotent, so running it twice is safe.

Fetch the script - it is public:

```bash
curl -O https://raw.githubusercontent.com/OmniTrustILM/helm-charts/2.18.0/charts/keycloak-internal/scripts/update_realm_from_2.17.0_to_2.18.0.py
```
Exec the script. *Default password used in appliance 2.17.0 and before was `admin`*. Change hostname if you are running on different hostname than `ilm.local`. In example we are using `--insecure` flag because the Kubernetes ingress certificate is self-signed.
```
$ python3 update_realm_from_2.17.0_to_2.18.0.py  --insecure
Enter the URL of the Keycloak (e.g., https://my.ilm.local/kc): https://ilm.local/kc/
Enter the admin username: admin
Enter the admin password:
Authenticated successfully.
Realm 'ILM' already exists. Verifying that all rebrand-related fields are migrated...
Default role already named 'default-roles-ilm', skipping.
Application client already renamed to 'ilm', skipping.
No audience mappers required updating on client 'ilm'.
Client 'account' URLs already migrated, skipping.
Client 'account-console' URLs already migrated, skipping.
Client 'security-admin-console' URLs already migrated, skipping.
Verification complete.
```
After executing this script you won't be able to access ILM anymore because realm has changed from `CZERTAINLY` to `ILM`. And other hand you have to execute this script before upgrading to 2.19.0 because it relies on running keycloak with the old realm.

## 3. Remove RKE2 together with ILM

Appliance 2.19.0 is using Traefik as the ingress controler as replacement for deprecated nginx ingress used before. Removing RKE2 makes transition to new RKE2 version 1.36 smoother.

Choose [Remove RKE2 & ILM](https://docs.otilm.com/docs/certificate-key/installation-guide/deployment/deployment-appliance/TUI/main-menu#remove-rke2--ilm) from advanced menu.

> All your data are living in the PostgreSQL database, so removing RKE2 and ILM will not delete them.

## 4. Update the Operating System & ilm-appliance-tools

Choose [Update Operating System](https://docs.otilm.com/docs/certificate-key/installation-guide/deployment/deployment-appliance/TUI/main-menu#update-operating-system) from the advanced menu. This runs `apt update && apt upgrade`, which is also how the new `ilm-appliance-tools` package - the one that installs RKE2 1.36 with Traefik - gets onto the appliance. Logout and log back in to apply changes, the TUI itself comes from that package.

> **Keep your own configuration.** The files in `/etc/ilm-ansible/vars/` are
> shipped as Debian conffiles, so `apt` asks about every one of them you have
> ever changed:
>
> ```
> Configuration file '/etc/ilm-ansible/vars/database.yml'
>  ==> Modified (by you or by a script) since installation.
>  ==> Package distributor has shipped an updated version.
>    What would you like to do about it ?  Your options are:
>     Y or I  : install the package maintainer's version
>     N or O  : keep your currently-installed version
> ```
>
> Answer **N**, keep your currently-installed version - it is the default, so
> pressing enter is enough. Taking the maintainer's version resets the database
> credentials, the registry credentials, the network settings and the list of
> installed ILM components to the defaults of a fresh appliance.

> **The database keeps its old name.** An appliance that started its life as
> CZERTAINLY has `czertainlydb` and `czertainlyuser` in `database.yml`. There is
> nothing to rename: ILM runs happily with those names for as long as they stay
> in the configuration, and the answer above is what keeps them there.

Verify that the new tools are installed - `ilm-versions` prints all of it in
one line, and at this point RKE2 and the chart are gone, so only the first two
numbers are interesting:

```bash
ilm@ilm:~$ ilm-versions
appliance: 2.17.0; tools: 2.19.0; chart: ?.?.?; rke2: not installed
```

The [documented](https://docs.otilm.com/docs/certificate-key/installation-guide/deployment/deployment-appliance/operations#versioning)
way of asking for the package itself works as well:

```bash
ilm@ilm:~$ apt -q show ilm-appliance-tools
Package: ilm-appliance-tools
Version: 2.19.0
...
```

> The version in the title bar of the TUI is read from `/etc/appliance_version`
> and it does **not** change during an upgrade - it records the Debian base the
> appliance was built from. The version of the tools is the one printed above,
> and it is perfectly fine for it to be higher than the version of the
> appliance, see the
> [versioning documentation](https://docs.otilm.com/docs/certificate-key/installation-guide/deployment/deployment-appliance/operations#versioning).

## 5. Registry name

During re-branding process the registry name has been migrated to new hostname. Old name `harbor.3key.company` is being abandoned tor the new name `hub.omnitrustregistry.com`.

From main menu choose [Docker Repository](https://docs.otilm.com/docs/certificate-key/installation-guide/deployment/deployment-appliance/TUI/main-menu#configure-docker-repository-access-credentials) and change hostname of the repository.

## 6. Run the installation of 2.18.0

Chart 2.19.0 could be installed on the 2.17.0 database directly - the
[operations documentation](https://docs.otilm.com/docs/certificate-key/installation-guide/deployment/deployment-appliance/operations#ilm-upgrade)
says that raising the version number and running the installation works for
anything from 2.8.0 upwards, and neither the official upgrade notes nor the
chart require an intermediate release. We still go through 2.18.0 first, so
that the database migrations of that release are applied and verified on their
own - if something goes wrong there is only one release to look at.

Set `version: 2.18.0` in main menu, under [Configure ILM](https://docs.otilm.com/docs/certificate-key/installation-guide/deployment/deployment-appliance/TUI/main-menu#configure-ilm)

Run [Install ILM](https://docs.otilm.com/docs/certificate-key/installation-guide/deployment/deployment-appliance/TUI/main-menu#install-ilm) from the main menu.

Wait for the installation to complete. Verify the installation by checking web interface of the ILM components.

## 7. Run the upgrade to 2.19.0

Set `version: 2.19.0` in main menu, under [Configure ILM](https://docs.otilm.com/docs/certificate-key/installation-guide/deployment/deployment-appliance/TUI/main-menu#configure-ilm)

Run [Install only ILM](https://docs.otilm.com/docs/certificate-key/installation-guide/deployment/deployment-appliance/TUI/advanced-menu#install-only-ilm) in advanced menu. RKE2 is already running at this point, so there is nothing to install besides ILM itself.

> Don't start any real work between step 6 and step 7. This one is a Helm
> upgrade of a running release, and 2.19.0 moves RabbitMQ from the `czertainly`
> virtual host to `/`, so anything still queued from the 2.18.0 installation
> stays behind in the old virtual host and is never processed. Once 2.19.0 is
> stable the old virtual host can be deleted from the RabbitMQ management UI.

Finally verify what is running:

```bash
ilm@ilm:~$ ilm-versions --detailed
ilm@ilm:~$ ilm-status
```

and log in to the administrator interface with your existing certificate - it
still works because of step 1.
