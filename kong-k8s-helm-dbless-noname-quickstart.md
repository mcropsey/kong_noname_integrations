# Install Kong with the Noname Plugin on Kubernetes (Helm, DB-less)

This guide installs one Kong gateway on Kubernetes with the Helm chart and turns on the Noname (API Security) plugin for all traffic.

- Kong 3.9.4, Helm chart `kong/kong` 3.4.1
- DB-less (no database), no Ingress Controller
- Namespace `kong`, Helm release `kong`

Tested on k3s v1.37 with Noname plugin 5.0.1.

---

## What you need

- A Kubernetes cluster and `kubectl` that can reach it
- `helm` v3 (Step 0 shows how to install it)
- The cluster can reach your API Security engine on the internet (HTTPS)
- Access to the API Security UI to download the plugin zip

Do every step on the machine where you run `kubectl`.

---

## Step 0: Install helm (skip if `helm version` already works)

```bash
cd /tmp
curl -LO https://get.helm.sh/helm-v3.22.1-linux-amd64.tar.gz
curl -LO https://get.helm.sh/helm-v3.22.1-linux-amd64.tar.gz.sha256sum
sha256sum -c helm-v3.22.1-linux-amd64.tar.gz.sha256sum     # must say OK
tar xzf helm-v3.22.1-linux-amd64.tar.gz
sudo install -m 755 linux-amd64/helm /usr/local/bin/helm
helm version
```

**k3s only:** helm can't find the cluster on its own. Run this, and add the same line to `~/.bashrc`:

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

If you skip it, helm fails with `Kubernetes cluster unreachable ... localhost:8080`.

---

## Step 1: Make a working folder

```bash
mkdir -p ~/kong-noname-k8s
cd ~/kong-noname-k8s
```

Stay in this folder for all the steps below.

---

## Step 2: Download the Noname plugin

1. In the API Security UI go to **Settings > Integrations > Traffic Source & Prevention**.
2. Click **Add Integration**, then the **Kong** tile.
3. Enter a name, pick the engine, click **Create**.
4. Download the **zip**, then click **Close**.

Copy the zip into `~/kong-noname-k8s` and unzip it:

```bash
unzip <downloaded-file>.zip -d noname-security-kong-policy
ls noname-security-kong-policy/kong/plugins/nonamesecurity
```

You should see 6 files: `batch_queue.lua configuration.lua handler.lua prevention.lua schema.lua utils.lua`.

`schema.lua` is made for your integration. It holds your engine URL and your **source key**, so don't commit it to git or share it. Check the engine URL:

```bash
grep NN_ENGINE_URL noname-security-kong-policy/kong/plugins/nonamesecurity/schema.lua
```

---

## Step 3: Load the plugin into the cluster

```bash
kubectl create namespace kong --dry-run=client -o yaml | kubectl apply -f -

cd noname-security-kong-policy/kong/plugins
kubectl create configmap kong-plugin-nonamesecurity \
  --from-file=nonamesecurity -n kong --save-config
cd ~/kong-noname-k8s
```

Check it. All 6 `.lua` files should be listed:

```bash
kubectl describe configmap kong-plugin-nonamesecurity -n kong | grep '\.lua'
```

---

## Step 4: Create the Kong config (`kong.yml`)

This file holds your services and routes, and turns on the Noname plugin. **Change the `url:` and `hosts:` lines to match your apps.**

```bash
cat > kong.yml <<'EOF'
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
  # Noname plugin, on for every service
  - name: nonamesecurity

  - name: rate-limiting
    service: crapi-app
    config:
      minute: 100
      hour: 2000
      policy: local

  - name: rate-limiting
    service: crapi-mail
    config:
      minute: 100
      hour: 2000
      policy: local
EOF
```

Load it into the cluster. The key name must be `kong.yml`:

```bash
kubectl create configmap kong-dbless-config \
  --from-file=kong.yml=kong.yml -n kong --save-config
```

---

## Step 5: Create the Helm values file

```bash
cat > values-noname.yaml <<'EOF'
image:
  repository: kong
  tag: "3.9.4"

replicaCount: 1

env:
  database: "off"

ingressController:
  enabled: false

dblessConfig:
  configMap: kong-dbless-config

plugins:
  configMaps:
    - name: kong-plugin-nonamesecurity
      pluginName: nonamesecurity

proxy:
  enabled: true
  type: LoadBalancer

admin:
  enabled: true
  type: ClusterIP
  http:
    enabled: true
  tls:
    enabled: false

manager:
  enabled: false
EOF
```

What this does: it runs stock Kong, mounts the plugin from Step 3, sets `KONG_PLUGINS=bundled,nonamesecurity`, and loads `kong.yml` from Step 4. The proxy listens on ports 80 and 443.

> No LoadBalancer in your cluster, or ports 80/443 are already used by another LoadBalancer? Change `type: LoadBalancer` to `type: NodePort`.

---

## Step 6: Install Kong

```bash
helm repo add kong https://charts.konghq.com
helm repo update
helm install kong kong/kong -n kong -f values-noname.yaml --version 3.4.1
kubectl rollout status deploy/kong-kong -n kong
```

Wait for `successfully rolled out`.

---

## Step 7: Check it works

**1. The pod is running:**

```bash
kubectl get pods -n kong
```

**2. The plugin is loaded.** The first command should print `bundled,nonamesecurity`, and the second should list the 6 files:

```bash
kubectl exec deploy/kong-kong -n kong -- printenv KONG_PLUGINS
kubectl exec deploy/kong-kong -n kong -- ls /opt/kong/plugins/nonamesecurity
```

**3. No errors in the log.** This should print `no errors`:

```bash
kubectl logs deploy/kong-kong -n kong | grep -E "\[error\]|\[crit\]|NONAME" || echo "no errors"
```

(Warnings like `rate-limiting: config.redis_... is deprecated` are normal. Ignore them.)

**4. Traffic goes through Kong.** Find the proxy address:

```bash
kubectl get svc kong-kong-proxy -n kong
```

Use an `EXTERNAL-IP` (or, with NodePort, `<node-ip>:<port>`). This should print `200`:

```bash
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: crapi.cropseyit.com" http://<EXTERNAL-IP>/
```

If you get `000` right after the install, wait 30 seconds and try again. The LoadBalancer needs a moment to start.

**5. The traffic shows up in the API Security UI** for your integration. It can take a minute.

Done.

---

## Changing kong.yml later

Edit `kong.yml`, then:

```bash
kubectl create configmap kong-dbless-config \
  --from-file=kong.yml=kong.yml -n kong \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl rollout restart deploy/kong-kong -n kong
kubectl rollout status deploy/kong-kong -n kong
```

Kong doesn't see the change until you restart it.

## Updating the plugin to a new version

1. Download a new zip from the UI (Step 2). Unzip it into a **new, empty** `noname-security-kong-policy` folder. Don't mix files from two downloads.
2. Load it and restart Kong:
   ```bash
   cd ~/kong-noname-k8s/noname-security-kong-policy/kong/plugins
   kubectl create configmap kong-plugin-nonamesecurity \
     --from-file=nonamesecurity -n kong \
     --dry-run=client -o yaml | kubectl apply -f -
   cd ~/kong-noname-k8s
   kubectl rollout restart deploy/kong-kong -n kong
   kubectl rollout status deploy/kong-kong -n kong
   ```
3. Check the version:
   ```bash
   kubectl exec deploy/kong-kong -n kong -- grep NN_VERSION /opt/kong/plugins/nonamesecurity/schema.lua
   ```

With 1 replica there's a short gap in traffic during the restart.

## Turning the plugin off (Kong keeps running)

Open `kong.yml`, delete the line `- name: nonamesecurity`, then run the commands in *Changing kong.yml later*.

## Uninstall everything

```bash
helm uninstall kong -n kong
kubectl delete configmap kong-plugin-nonamesecurity kong-dbless-config -n kong
```

Then delete the integration in the API Security UI.

---

## Troubleshooting

| Problem | What to do |
| --- | --- |
| `helm`: `Kubernetes cluster unreachable ... localhost:8080` | k3s: `export KUBECONFIG=/etc/rancher/k3s/k3s.yaml` (Step 0). |
| Pod stuck in `ContainerCreating` | A ConfigMap is missing. `kubectl describe pod -n kong -l app.kubernetes.io/instance=kong` names it. Redo Step 3 or 4. |
| Pod in `CrashLoopBackOff` | `kubectl logs deploy/kong-kong -n kong --previous`. Usually a typo in `kong.yml`. Fix it, then follow *Changing kong.yml later*. |
| `EXTERNAL-IP` stays `<pending>` | Your cluster has no LoadBalancer, or 80/443 is already used. Set `type: NodePort` in `values-noname.yaml`, then `helm upgrade kong kong/kong -n kong -f values-noname.yaml --version 3.4.1`. |
| `plugin 'nonamesecurity' not found` | The ConfigMap name in Step 3 doesn't match `values-noname.yaml`. Both must be `kong-plugin-nonamesecurity`. |
| Log shows `[NONAME] Failed to connect` or `returned with an error` | Kong can't reach the engine, or the engine rejects the source key. Test from the cluster (see below). If the test returns 200, download a fresh zip and do *Updating the plugin*. |
| Traffic works, but nothing in the UI and no errors | Check that `kong.yml` has `- name: nonamesecurity` under `plugins:`. Test the engine (see below). |

**Test the engine from inside the cluster.** The Kong image has no `curl`, so use a temporary pod. Use the host from your `NN_ENGINE_URL` (Step 2). It should print `200`:

```bash
kubectl run nn-test -n kong --rm -i --restart=Never --image=curlimages/curl \
  --command -- curl -s -o /dev/null -w "%{http_code}\n" https://<your-engine-host>/engine
```

**No `[NONAME]` lines in the log is normal.** The plugin only logs when something is wrong.
