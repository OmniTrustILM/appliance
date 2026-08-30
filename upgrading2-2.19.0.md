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

## 1. Keep trusting your current root CA

> This section matter only if you were using pre-provisioned certificates. Feel free to skip it if you were using your own CA.

Chart 2.18.0 replaced the built-in root certificate authority. Old appliances trust [`CN=CZERTAINLY Dummy Root CA`](https://github.com/OmniTrustILM/helm-charts/blob/2.17.0/dummy-certificates/certs/root-ca.cert.pem); 2.18.0 and later ship [`CN=Dummy Root CA`](https://github.com/OmniTrustILM/helm-charts/blob/2.19.0/dummy-certificates/certs/root-ca.cert.pem). The admin's certificate were with pre 2.19.0 apliance was issued by the old root, you can continue to use it as long as both the old and new root CAs are trusted.

To ensure continued trust, you need to combine the [old](https://github.com/OmniTrustILM/helm-charts/blob/2.17.0/dummy-certificates/certs/root-ca.cert.pem) and [new](https://github.com/OmniTrustILM/helm-charts/blob/2.19.0/dummy-certificates/certs/root-ca.cert.pem) root CAs into a single file. Configure both of them as [custom trusted certificate](https://docs.otilm.com/docs/certificate-key/installation-guide/deployment/deployment-appliance/TUI/main-menu#configure-custom-trusted-certificates).

You don't need to re-run Install, because in the later steps we are goin to upgrade the appliance using the new RKE2 and Traefik setup.

## 2. Migrate the Keycloak realm (needed by chart 2.18.0)

> Only for appliances that were installed as CZERTAINLY 2.17.0 and older.
The realm is renamed in place, keeping every UUID, user, session and group
membership; the script is idempotent, so running it twice is safe.

Fetch the script - it is public:

```bash
curl -O https://raw.githubusercontent.com/OmniTrustILM/helm-charts/main/charts/keycloak-internal/scripts/update_realm_from_2.17.0_to_2.18.0.py
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

Choose [Update Operating System](https://docs.otilm.com/docs/certificate-key/installation-guide/deployment/deployment-appliance/TUI/main-menu#update-operating-system) from the advanced menu. And make sure you are have installed new version ilm-appliance-tools (TODO: howto). Logout and log back in to apply changes.

> All your data are living in the PostgreSQL database, so removing RKE2 and ILM will not delete them.

## 4. Run the installation of 2.18.0

Set `version: 2.18.0` in main menu, under [Configure ILM](https://docs.otilm.com/docs/certificate-key/installation-guide/deployment/deployment-appliance/TUI/main-menu#configure-ilm)

Run [Install ILM](https://docs.otilm.com/docs/certificate-key/installation-guide/deployment/deployment-appliance/TUI/main-menu#install-ilm) from the main menu.

Wait for the installation to complete. Verify the installation by checking web interface of the ILM components.

## 5. Run the upgrade to 2.19.0

Set `version: 2.19.0` in main menu, under [Configure ILM](https://docs.otilm.com/docs/certificate-key/installation-guide/deployment/deployment-appliance/TUI/main-menu#configure-ilm)

Run [Install only ILM](https://docs.otilm.com/docs/certificate-key/installation-guide/deployment/deployment-appliance/TUI/advanced-menu#install-only-ilm) in advanced menu.
