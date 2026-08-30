# Upgrading VA to version 2.19.0

Between appliance 2.17.0 and 2.19.0 the ILM Helm chart went through **two**
releases that need manual work, 2.18.0 and 2.19.0. The chart documentation
calls the 2.18.0 steps *"Manual steps required when upgrading from any earlier
version"*, so **they are not skipped by jumping straight to the 2.19.0 chart** -
they have to be performed on the way. Read the official
[Helm upgrade notes](https://docs.otilm.com/docs/certificate-key/installation-guide/deployment/deployment-helm/upgrading)
alongside this document; what follows is the appliance-specific version of it.

Three things bite in this upgrade, and each one can lock you out or lose data
if it is done in the wrong order:

| what | comes from | symptom if skipped |
|---|---|---|
| the trusted root CA changed name | chart 2.18.0 | administrators cannot log in, the browser fails the TLS handshake |
| the Keycloak realm was renamed CZERTAINLY -> ILM | chart 2.18.0 | Keycloak fails to start, `--import-realm` hits a duplicate primary key |
| the RabbitMQ virtual host and exchanges were renamed | chart 2.19.0 | messages left in the old vhost are never processed |

> **Order matters.** The Keycloak migration runs **before** `helm upgrade`,
> while the old Keycloak is still running. So does the RabbitMQ drain.

## 0. Back up first

1. A snapshot of the whole virtual machine, so you can go back.
2. A database dump, see the
   [operations documentation](https://docs.otilm.com/docs/certificate-key/installation-guide/deployment/deployment-appliance/operations#backup).
3. A copy of `/etc/ilm-ansible/vars/` - it holds `keycloak.yml` (the Keycloak
   client secret), `cbom.yml` (the object storage credentials) and everything
   the TUI has configured.
4. If the CBOM repository is installed, a copy of `/var/lib/ilm/cbom-repo`.
   That data lives outside the database.

## 1. Keep trusting your current root CA

Chart 2.18.0 replaced the built-in root certificate authority. Old appliances
trust `CN=CZERTAINLY Dummy Root CA`; 2.18.0 and later ship
`CN=Dummy Root CA`. The certificate in your browser was issued by the old one.

That certificate authority is used in two places, which is why the failure
looks like an outage rather than a login error:

* `core` validates the client certificate chain against it,
* the Traefik `TLSOption` uses it for `clientAuth`, so **the TLS handshake
  itself** is refused before any page is served.

Put both authorities in one file **before** upgrading:

```bash
# the CA this appliance trusts today
kubectl -n ilm get secret trusted-certificates -o jsonpath='{.data.ca\.crt}' \
    | base64 -d > /home/ilm/trustedCA.pem

# add the one the new chart brings
helm show values oci://hub.omnitrustregistry.com/ilm-helm/ilm --version 2.19.0 |
    python3 -c 'import sys,yaml; print(yaml.safe_load(sys.stdin)["trusted"]["certificates"])' \
    >> /home/ilm/trustedCA.pem

# there must be two self-signed roots in the file now
openssl crl2pkcs7 -nocrl -certfile /home/ilm/trustedCA.pem |
    openssl pkcs7 -print_certs -noout
```

and point the playbook at it in `/etc/ilm-ansible/vars/trustedCA.yml`:

```yaml
trustedCA_file: /home/ilm/trustedCA.pem
```

> `trustedCA_file` **replaces** the certificates the chart ships, it does not
> add to them. A file containing only the old authority repairs the old
> certificates and breaks the new ones, so keep both in it.

A better long-term answer is to stop using the built-in "dummy" authority
altogether: issue the administrator certificate from a CA you control and put
that CA in `trustedCA.yml`. Then a rename in the chart cannot affect you.

## 2. Migrate the Keycloak realm (needed by chart 2.18.0)

Only for appliances that were installed as CZERTAINLY, i.e. 2.17.0 and older.
The realm is renamed in place, keeping every UUID, user, session and group
membership; the script is idempotent, so running it twice is safe.

Fetch the script - it is public:

```bash
curl -O https://raw.githubusercontent.com/OmniTrustILM/helm-charts/main/charts/keycloak-internal/scripts/update_realm_from_2.17.0_to_2.18.0.py
```

If the appliance is older than 2.17.0, run
`update_realm_from_2.7.0_to_2.14.0.py` and
`update_realm_from_2.14.0_to_2.17.0.py` first, in that order.

Keycloak is not published through the appliance ingress, so give the script a
way in and read the administrator credentials out of the cluster:

```bash
kubectl -n ilm port-forward svc/keycloak-internal-service 8080:8080 &

kubectl -n ilm get secret keycloak-internal-secret \
    -o jsonpath='{.data.username}' | base64 -d; echo
kubectl -n ilm get secret keycloak-internal-secret \
    -o jsonpath='{.data.password}' | base64 -d; echo
```

Then run it. It asks for the URL and those credentials interactively:

```bash
python3 update_realm_from_2.17.0_to_2.18.0.py
# Enter the URL of the Keycloak (e.g., https://my.ilm.local/kc): http://127.0.0.1:8080
# Enter the admin username: ...
# Enter the admin password: ...
```

`python3-requests`, which the script needs, is already installed on the
appliance. Expect `Renamed realm 'CZERTAINLY' -> 'ILM' ...` followed by
`Migration complete.` Stop the port-forward afterwards.

## 3. Delete the resources the upgrade cannot patch (needed by chart 2.18.0)

Three objects cannot be upgraded in place - a Deployment's `spec.selector` is
immutable, and the ingress admission webhook rejects a duplicate host and path.
Helm recreates all of them under their new names.

```bash
kubectl -n ilm get deploy | grep -E 'core|pg-bouncer'
kubectl -n ilm get ingress

kubectl -n ilm delete deployment <old core deployment>
kubectl -n ilm delete deployment <old pg-bouncer deployment>
kubectl -n ilm delete ingress <old ingress>
```

List them first rather than copying names from here: on an appliance installed
as CZERTAINLY they carry the old prefix, while a 2.19.0 install has
`core-deployment`, `pg-bouncer-deployment` and `ilm-platform-core`.

From 2.18.0 on the umbrella chart sets `nameOverride: platform-core`, which
decouples these names from the chart name - so this deletion step is a one-off
and later upgrades will not need it. Do not override that value.

## 4. RabbitMQ virtual host and exchange rename (chart 2.19.0)

The messaging defaults moved:

| parameter | old | new |
|---|---|---|
| `messaging.virtualHost` | `czertainly` | `/` |
| `bootstrap.exchange` | `czertainly` | `ilm` |
| `bootstrap.proxy.exchange` | `czertainly-proxy` | `ilm-proxy` |

The appliance does not override any of them, so it takes the new defaults. The
provisioning service creates the new exchanges and queues on the `/` virtual
host by itself; the old `czertainly` virtual host stays behind, orphaned but
not deleted, **with whatever messages were still in it**.

If losing those messages is acceptable, upgrade and delete the old virtual host
from the RabbitMQ management UI once the platform is stable.

If it is not, drain first - before the upgrade:

```bash
kubectl -n ilm scale deployment api-gateway-deployment --replicas=0
```

then watch the queue depths on the old virtual host until they reach zero,
using the drain script in the
[official upgrade notes](https://docs.otilm.com/docs/certificate-key/installation-guide/deployment/deployment-helm/upgrading),
and only then continue.

## 5. Run the upgrade

The appliance does not call `helm upgrade` by hand - the playbook owns the
values file and the release. Install the new tools package and raise the
version:

```bash
sudo apt update
sudo apt install ilm-appliance-tools
```

Set `version: 2.19.0` under `ilm:` in `/etc/ilm-ansible/vars/ilm.yml`, either
in the ILM screen of `ilm-tui` or by editing the file, then:

```bash
sudo ansible-playbook /etc/ilm-ansible/playbooks/ilm.yml --tags ilm
```

If the api-gateway was scaled to zero for the drain, the upgrade scales it back
up on its own.

## 6. Verify

```bash
ilm-versions --detailed     # chart: ilm-2.19.0, and every release listed
ilm-status                  # all three namespaces reporting PODs OK
```

Then log in to `https://<your appliance>/administrator/` with your existing
administrator certificate. It still works because of step 1.

## If you are already locked out

The Traefik `TLSOption` uses `VerifyClientCertIfGiven`, so a client that offers
**no** certificate still completes the handshake. Decline the certificate
prompt in the browser and the interface is reachable, and Keycloak login works
if it is enabled.

To repair the trust from the console or over ssh, build the two-certificate
file as in step 1, then:

```bash
kubectl -n ilm create secret generic trusted-certificates \
    --from-file=ca.crt=/home/ilm/trustedCA.pem --dry-run=client -o yaml |
    kubectl -n ilm apply -f -
kubectl -n ilm rollout restart deploy/core-deployment
```

Traefik watches the secret, so the handshake recovers by itself. This edit is
undone by the next `helm upgrade`, so set `trustedCA_file` in
`/etc/ilm-ansible/vars/trustedCA.yml` as well and re-run the playbook to make
it permanent.
