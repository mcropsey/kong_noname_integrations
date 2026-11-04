# Noname Kong Plugin: Install (Kubernetes, Helm, DB-less, single instance)

**Target:** one Kong gateway pod in a Kubernetes cluster, deployed with the `kong/kong` Helm chart
**Mode:** DB-less (`env.database: "off"`), no Kong Ingress Controller, `replicaCount: 1`
**Namespace:** `kong` · **Helm release:** `kong` → Deployment `kong-kong`
**Method:** the documented *Kubernetes: Helm Chart* install (plugin delivered as a ConfigMap that the chart mounts) combined with the documented *DB-less* association (plugin declared in `kong.yml`).

This guide is self-contained: the two files it needs (`kong.yml` and `values-noname.yaml`) are created by commands in Steps 3 and 4. The only thing you bring is the Noname zip from Step 1.

Prerequisites: `kubectl` pointed at the cluster, `helm` v3, and the cluster able to reach the API Security engine URL (outbound HTTPS).

**Where to run the commands.** Run everything on the machine where you use `kubectl` and `helm`, from one working folder:

```bash
mkdir -p ~/kong-noname-k8s && cd ~/kong-noname-k8s
```

Stay in this folder for every step. Unzip the Noname download here too (Step 1), so the paths in Step 2 work as written.

---

## Step 1: Create the integration profile and download the plugin

1. In the API Security UI go to **Settings > Integrations > Traffic Source & Prevention**.
2. **Add Integration** → **Kong** tile → name the profile → select the engine.
3. (Optional) **Advanced Settings** to configure the original host/path.
4. **Create** → download the **zip** → **Close**.

Unzip it into this folder (`unzip <downloaded-file>.zip -d noname-security-kong-policy`). The folder you need:

```
noname-security-kong-policy/kong/plugins/nonamesecurity/
├── batch_queue.lua   ├── handler.lua     ├── schema.lua
├── configuration.lua ├── prevention.lua  └── utils.lua
```

`schema.lua` comes pre-filled for this profile (engine URL, source index, source key). Use this copy, never a generic one. Check it:

```bash
grep -E "NN_ENGINE_URL|NN_SOURCE_KEY" noname-security-kong-policy/kong/plugins/nonamesecurity/schema.lua
```

## Step 2: Install the code (plugin ConfigMap)

Create the namespace and load the plugin's Lua files into a ConfigMap:

```bash
kubectl create namespace kong

cd noname-security-kong-policy/kong/plugins
kubectl create configmap kong-plugin-nonamesecurity \
  --from-file=nonamesecurity -n kong
cd -
```

Verify all six `.lua` files are keys in the ConfigMap:

```bash
kubectl get configmap kong-plugin-nonamesecurity -n kong -o jsonpath='{.data}' | tr ',' '\n' | grep -o '"[a-z_]*\.lua"'
```

The plugin folder is flat (no subdirectories), so a single ConfigMap holds it. The chart mounts it at `/opt/kong/plugins/nonamesecurity` and sets `KONG_LUA_PACKAGE_PATH` so Kong finds it.

## Step 3: Associate the plugin (DB-less: `kong.yml` ConfigMap)

DB-less Kong rejects Admin API writes, so the documented `POST /services/<service>/plugins` call is replaced by a declarative entry in `kong.yml`.

Create `kong.yml` (this command writes the file). It holds your crAPI routes plus the plugin. **Before pasting, check the two `url:` lines**: if crAPI runs inside the cluster, replace the IPs with its Service DNS names (for example `http://crapi-web.crapi.svc.cluster.local:80`).

```bash
cat > kong.yml <<'EOF'
# Kong DB-less declarative config WITH the Noname plugin.
# Loaded into the cluster as ConfigMap "kong-dbless-config" (key: kong.yml).

_format_version: "3.0"
_transform: true

services:
  - name: crapi-app
    url: http://192.168.1.101:8888
    routes:
      - name: crapi-root
        hosts:
          - crapi.cropseyit.com
        paths:
          - /
        strip_path: false
        preserve_host: true

  - name: crapi-mail
    url: http://192.168.1.101:8025
    routes:
      - name: crapi-mailhog
        hosts:
          - mail.cropseyit.com
        paths:
          - /
        strip_path: false
        preserve_host: true

plugins:
  # Noname Security: global, applies to every service.
  - name: nonamesecurity

  - name: rate-limiting
    service: crapi-app
    config:
      minute: 100
      hour: 2000
      policy: local
      limit_by: ip
      fault_tolerant: true
      hide_client_headers: false

  - name: rate-limiting
    service: crapi-mail
    config:
      minute: 100
      hour: 2000
      policy: local
      limit_by: ip
      fault_tolerant: true
      hide_client_headers: false
EOF
```

The Noname part is the single `- name: nonamesecurity` entry under `plugins:`. It is global (all services); to cover one service, add `service: crapi-app` under it. If you already have a `kong.yml`, you can use it instead and just add that entry.

Load it as a ConfigMap. The chart requires the key to be **`kong.yml`**:

```bash
kubectl create configmap kong-dbless-config \
  --from-file=kong.yml=kong.yml -n kong
```

Both ConfigMaps (Steps 2 and 3) must exist **before** the Helm install, or the pod can't start.

## Step 4: Enable the plugin and deploy (Helm)

Create `values-noname.yaml` (this command writes the file):

```bash
cat > values-noname.yaml <<'EOF'
image:
  repository: kong
  tag: "3.6"            # match the Kong version you want to run

replicaCount: 1         # single Kong instance

env:
  database: "off"       # DB-less

ingressController:
  enabled: false        # gateway only, config comes from kong.yml

dblessConfig:
  configMap: kong-dbless-config

plugins:
  configMaps:
    - name: kong-plugin-nonamesecurity
      pluginName: nonamesecurity

proxy:
  enabled: true
  type: LoadBalancer    # NodePort if no LoadBalancer is available

admin:
  enabled: true
  type: ClusterIP
  http:
    enabled: true
  tls:
    enabled: false
EOF
```

What the chart does with this:
- `plugins.configMaps` → mounts the plugin and sets **`KONG_PLUGINS=bundled,nonamesecurity`**. You don't set `KONG_PLUGINS` yourself, and `bundled` is kept, so `rate-limiting` still works.
- `dblessConfig.configMap` + `database: "off"` + controller disabled → mounts `kong.yml` at `/kong_dbless/kong.yml` and sets `KONG_DECLARATIVE_CONFIG`.

Install:

```bash
helm repo add kong https://charts.konghq.com
helm repo update
helm install kong kong/kong -n kong -f values-noname.yaml
```

## Step 5: Verify

```bash
kubectl get pods -n kong
kubectl logs deploy/kong-kong -n kong -c proxy | tail -20

kubectl exec deploy/kong-kong -n kong -c proxy -- printenv KONG_PLUGINS
#   bundled,nonamesecurity
kubectl exec deploy/kong-kong -n kong -c proxy -- ls /opt/kong/plugins/nonamesecurity

kubectl port-forward deploy/kong-kong -n kong 8001:8001 &
curl -s localhost:8001/plugins/enabled | grep nonamesecurity
curl -s localhost:8001/plugins | grep -o '"name":"nonamesecurity"'
```

Get the proxy address and send traffic:

```bash
kubectl get svc kong-kong-proxy -n kong
curl -H "Host: crapi.cropseyit.com" http://<EXTERNAL-IP-or-NODE:PORT>/
```

Then confirm the traffic shows up in the API Security UI.

---

## Configuring the plugin after install

Most runtime behavior is managed in the API Security UI and fetched by the plugin periodically, so it doesn't need a redeploy:
**Settings > Integrations > Traffic Source & Prevention → ⋮ > Edit → Advanced Settings**

- Connection / Send / Read Timeout, Max Body Size, Batch Size, Support chunked traffic
- **Support Prevention** (on by default) and its header name, status code, and rule-clear retries
- Configuration Fetch Interval / Timeout

Anything else is fixed in `schema.lua`. To change it, edit `schema.lua` locally, update the plugin ConfigMap, and restart (same commands as *Updating*, below).

## Changing kong.yml (routes, services, plugins)

```bash
kubectl create configmap kong-dbless-config \
  --from-file=kong.yml=kong.yml -n kong \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl rollout restart deploy/kong-kong -n kong
```

The chart only adds a config checksum (automatic restart) when the config is inline in `values.yaml`. With an external ConfigMap you must restart it yourself.

---

## Updating to a new plugin version

1. Re-download the zip from the UI (the `schema.lua` is regenerated for your profile) and unzip it. Never merge files from different downloads.
2. Update the ConfigMap in place:
   ```bash
   cd noname-security-kong-policy/kong/plugins
   kubectl create configmap kong-plugin-nonamesecurity \
     --from-file=nonamesecurity -n kong \
     --dry-run=client -o yaml | kubectl apply -f -
   cd -
   ```
3. Restart the pod so the new Lua loads. A ConfigMap change alone doesn't reload running workers:
   ```bash
   kubectl rollout restart deploy/kong-kong -n kong
   kubectl rollout status deploy/kong-kong -n kong
   ```
4. Confirm: `kubectl exec deploy/kong-kong -n kong -c proxy -- grep NN_VERSION /opt/kong/plugins/nonamesecurity/schema.lua`

No re-association needed; the `kong.yml` entry stays as is.

> With `replicaCount: 1`, a restart causes a brief gap while the new pod starts. For zero downtime, scale to 2+ replicas so the rollout replaces pods one at a time.

## Rollback

Keep the previous plugin folder (or zip) and `kong.yml`. Re-apply them with the update commands above, then `kubectl rollout restart deploy/kong-kong -n kong`.

To remove only the plugin from traffic (keeps the code installed):

```bash
sed -i.bak '/- name: nonamesecurity/{N;/\n    service: crapi-app$/d;s/^.*\n//;}' kong.yml
kubectl create configmap kong-dbless-config \
  --from-file=kong.yml=kong.yml -n kong \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl rollout restart deploy/kong-kong -n kong
```

## Uninstall

1. Remove the plugin entry from `kong.yml` and the `plugins:` block from `values-noname.yaml` (`-i.bak` works with both macOS and Linux `sed` and keeps a backup):
   ```bash
   sed -i.bak '/- name: nonamesecurity/{N;/\n    service: crapi-app$/d;s/^.*\n//;}' kong.yml
   sed -i.bak '/^plugins:/,/pluginName: nonamesecurity/d' values-noname.yaml
   grep -n nonamesecurity kong.yml values-noname.yaml || echo "plugin removed"
   ```
2. Apply both, then delete the plugin ConfigMap:
   ```bash
   kubectl create configmap kong-dbless-config \
     --from-file=kong.yml=kong.yml -n kong \
     --dry-run=client -o yaml | kubectl apply -f -
   helm upgrade kong kong/kong -n kong -f values-noname.yaml
   kubectl delete configmap kong-plugin-nonamesecurity -n kong
   ```
   (Or remove Kong entirely: `helm uninstall kong -n kong`.)
3. Delete the integration profile in the API Security UI. Previously collected data stays.

---

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| Pod stuck in `ContainerCreating` | A ConfigMap is missing: `kubectl describe pod -n kong -l app.kubernetes.io/instance=kong` will name it. Create it (Steps 2–3). |
| Pod `CrashLoopBackOff` | `kubectl logs deploy/kong-kong -n kong -c proxy --previous`. Usually YAML in `kong.yml`, or the ConfigMap key isn't `kong.yml`. |
| `plugin 'nonamesecurity' not found` / not enabled | Check `printenv KONG_PLUGINS` and `ls /opt/kong/plugins/nonamesecurity` in the pod (Step 5). If empty, `plugins.configMaps` in the values doesn't match the ConfigMap name. |
| Kong uses an empty/default config | `ingressController.enabled` is still true, or `env.database` isn't `"off"`. The chart mounts `kong.yml` only when both are set. |
| Edited `kong.yml` or `schema.lua`, nothing changed | Update the ConfigMap **and** `kubectl rollout restart deploy/kong-kong -n kong`. |
| Loads fine, nothing in the UI | Plugin entry missing from `kong.yml`, or the pod can't reach the engine: `kubectl exec deploy/kong-kong -n kong -c proxy -- curl -sS -o /dev/null -w '%{http_code}\n' https://michaelc-lab.nonamesec.com/engine` (check egress NetworkPolicies / proxies). |
| No `[NONAME]` log lines | Needs **both** `NN_DEBUG_ENABLED = true` in `schema.lua` (update the ConfigMap) **and** `env.log_level: info` in `values-noname.yaml` (`helm upgrade`). Then `kubectl logs -f deploy/kong-kong -n kong -c proxy \| grep NONAME`. |

---

## Notes

**Why a ConfigMap instead of a custom image.** This is the method the Helm article documents. The stock `kong` image is used unchanged and the plugin travels with the cluster config. Building a custom image (like the Docker guide) also works with Helm: set `image.repository/tag` to your image, drop `plugins.configMaps`, and add `plugins.preInstalled: nonamesecurity`.

**Single instance.** `replicaCount: 1` gives exactly one Kong pod. Each Kong pod runs its own copy of the plugin and batches its own traffic, so scaling up later needs no Noname changes.

**Prevention is on by default.** The plugin fetches prevention rules and can block requests inline (fail-open after a 50 ms check timeout). For inspection only, turn off **Support Prevention** in the integration's Advanced Settings in the UI. No redeploy is needed.

**Plugin order.** Inside Kong, `nonamesecurity` runs at execution priority **2501**, higher than `rate-limiting` (~910). It sees requests **before** the rate limiter, so requests later rejected with 429 are still inspected. (The "priority 1" shown in API Security is the integration priority, a separate concept.)

**Traffic inspection is asynchronous.** Records are batched and sent in the background. Defaults: 1 MB payload cap per request/response, 500 ms engine timeout, no retries.

## Scope

Specific to: `kong/kong` Helm chart as a standalone gateway, DB-less, no Ingress Controller, single replica, self-managed control plane. Using the Kong Ingress Controller or Kong Operator (CRDs / `KongPlugin` / `KongClusterPlugin`) is a different method; see the *Kubernetes: Kong Operator* article. Konnect hybrid data planes: create the plugin ConfigMap as in Step 2, then follow the Konnect Hybrid article (the schema must also be registered in Konnect).
