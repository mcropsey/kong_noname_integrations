# Kong on Kubernetes (Helm, DB-less) + Noname Plugin

This guide has two parts:

- **Part 1:** install Kong on Kubernetes with the Kong Helm chart (DB-less, no database)
- **Part 2:** add the Noname (API Security) plugin to that Kong, following Noname's *Kong Plugin - Kubernetes Helm* doc

Kong keeps working after every step, so you can stop between the two parts.

Versions: Kong 3.9.4, Helm chart `kong/kong` 3.4.1. Tested on k3s v1.37 with Noname plugin 5.0.1.

---

## What you need

- A Kubernetes cluster and `kubectl` that can reach it
- `helm` v3 (Step 1 shows how to install it)
- For Part 2: the cluster can reach your API Security engine on the internet (HTTPS), and you can log in to the API Security UI

Do every step on the machine where you run `kubectl`.

---

# Part 1: Install Kong

## Step 1: Install helm (skip if `helm version` already works)

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

## Step 2: Make a working folder

```bash
mkdir -p ~/kong-k8s
cd ~/kong-k8s
```

Stay in this folder for all the steps below.

## Step 3: Create the Kong config (`kong.yml`)

This file holds your services and routes. **Change the `url:` and `hosts:` lines to match your apps**, and add one block per app.

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
    plugins:
      - name: rate-limiting
        config:
          minute: 1000
          hour: 20000
          limit_by: ip
          policy: local

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
      - name: rate-limiting
        config:
          minute: 1000
          hour: 20000
          limit_by: ip
          policy: local
EOF
```

Load it into the cluster. The key name must be `kong.yml`:

```bash
kubectl create namespace kong --dry-run=client -o yaml | kubectl apply -f -
kubectl create configmap kong-dbless-config \
  --from-file=kong.yml=kong.yml -n kong --save-config
```

## Step 4: Create the Helm values file

```bash
cat > values.yaml <<'EOF'
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

This runs stock Kong, reads `kong.yml` from Step 3, and listens on ports 80 and 443.

> No LoadBalancer in your cluster, or ports 80/443 are already used by another LoadBalancer? Change `type: LoadBalancer` to `type: NodePort`.

## Step 5: Install Kong

```bash
helm repo add kong https://charts.konghq.com
helm repo update
helm install kong kong/kong -n kong -f values.yaml --version 3.4.1
kubectl rollout status deploy/kong-kong -n kong
```

Wait for `successfully rolled out`.

## Step 6: Check Kong works

```bash
kubectl get pods -n kong
kubectl get svc kong-kong-proxy -n kong
```

The pod should be `Running`. Use an `EXTERNAL-IP` from the second command (or, with NodePort, `<node-ip>:<port>`). This should print `200`:

```bash
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: crapi.cropseyit.com" http://<EXTERNAL-IP>/
```

If you get `000` right after the install, wait 30 seconds and try again. The LoadBalancer needs a moment to start.

Kong is installed. Part 2 adds Noname.

---

# Part 2: Add the Noname plugin

## Step 7: Download the plugin

1. In the API Security UI go to **Settings > Integrations > Traffic Source & Prevention**.
2. Click **Add Integration**, then the **Kong** tile.
3. Enter a name, pick the engine, click **Create**.
4. Download the **zip**, then click **Close**.

Copy the zip into `~/kong-k8s` and unzip it:

```bash
cd ~/kong-k8s
unzip <downloaded-file>.zip -d noname-security-kong-policy
ls noname-security-kong-policy/kong/plugins/nonamesecurity
```

You should see 6 files: `batch_queue.lua configuration.lua handler.lua prevention.lua schema.lua utils.lua`.

`schema.lua` is made for your integration. It holds your engine URL and your **source key**, so don't commit it to git or share it. Check the engine URL:

```bash
grep NN_ENGINE_URL noname-security-kong-policy/kong/plugins/nonamesecurity/schema.lua
```

## Step 8: Load the plugin into the cluster

```bash
cd ~/kong-k8s/noname-security-kong-policy/kong/plugins
kubectl create configmap kong-plugin-nonamesecurity \
  --from-file=nonamesecurity -n kong --save-config
cd ~/kong-k8s
```

Check it. All 6 `.lua` files should be listed:

```bash
kubectl describe configmap kong-plugin-nonamesecurity -n kong | grep '\.lua'
```

## Step 9: Enable the plugin in Kong

Add the plugin to the values file:

```bash
cat >> values.yaml <<'EOF'

plugins:
  configMaps:
    - name: kong-plugin-nonamesecurity
      pluginName: nonamesecurity
EOF
```

Apply it. Kong restarts with the plugin loaded:

```bash
helm upgrade kong kong/kong -n kong -f values.yaml --version 3.4.1
kubectl rollout status deploy/kong-kong -n kong
```

Wait until only one Kong pod is left (the old one takes about 30 seconds to stop), so the checks below look at the new pod:

```bash
while [ $(kubectl get pods -n kong --no-headers | wc -l) -gt 1 ]; do sleep 5; done
kubectl get pods -n kong
```

Check. The first command should print `bundled,nonamesecurity`, and the second should list the 6 files:

```bash
kubectl exec deploy/kong-kong -n kong -- printenv KONG_PLUGINS
kubectl exec deploy/kong-kong -n kong -- ls /opt/kong/plugins/nonamesecurity
```

The plugin is loaded but not yet looking at traffic. Step 10 turns it on.

## Step 10: Turn the plugin on for all traffic

Add the plugin to `kong.yml`:

```bash
cat >> kong.yml <<'EOF'

plugins:
  - name: nonamesecurity
EOF
```

> If your `kong.yml` already has a `plugins:` line at the **left edge** (not indented), don't run the command above. Open `kong.yml` and add `  - name: nonamesecurity` (two spaces in front) on the line under that `plugins:`.

Load the new `kong.yml` and restart Kong:

```bash
kubectl create configmap kong-dbless-config \
  --from-file=kong.yml=kong.yml -n kong \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl rollout restart deploy/kong-kong -n kong
kubectl rollout status deploy/kong-kong -n kong
```

## Step 11: Check Noname works

Wait until only one Kong pod is left (the old one takes about 30 seconds to stop), so the checks below look at the new pod:

```bash
while [ $(kubectl get pods -n kong --no-headers | wc -l) -gt 1 ]; do sleep 5; done
kubectl get pods -n kong
```

**1. No errors in the log.** This should print `no errors`:

```bash
kubectl logs deploy/kong-kong -n kong | grep -E "\[error\]|\[crit\]|NONAME" || echo "no errors"
```

(Warnings like `rate-limiting: config.redis_... is deprecated` are normal. Ignore them.)

**2. Traffic still goes through Kong.** This should print `200`:

```bash
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: crapi.cropseyit.com" http://<EXTERNAL-IP>/
```

**3. The traffic shows up in the API Security UI** for your integration. It can take a few minutes.

Done.

---

## Changing kong.yml later

Edit `kong.yml`, then:

```bash
cd ~/kong-k8s
kubectl create configmap kong-dbless-config \
  --from-file=kong.yml=kong.yml -n kong \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl rollout restart deploy/kong-kong -n kong
kubectl rollout status deploy/kong-kong -n kong
```

Kong doesn't see the change until you restart it.

## Updating the Noname plugin to a new version

1. Download a new zip from the UI (Step 7). Unzip it into a **new, empty** `noname-security-kong-policy` folder. Don't mix files from two downloads.
2. Load it and restart Kong:
   ```bash
   cd ~/kong-k8s/noname-security-kong-policy/kong/plugins
   kubectl create configmap kong-plugin-nonamesecurity \
     --from-file=nonamesecurity -n kong \
     --dry-run=client -o yaml | kubectl apply -f -
   cd ~/kong-k8s
   kubectl rollout restart deploy/kong-kong -n kong
   kubectl rollout status deploy/kong-kong -n kong
   ```
3. Check the version:
   ```bash
   kubectl exec deploy/kong-kong -n kong -- grep NN_VERSION /opt/kong/plugins/nonamesecurity/schema.lua
   ```

With 1 replica there's a short gap in traffic during each restart.

## Removing Noname (Kong keeps running)

Do these in order:

1. Open `kong.yml`, delete the `- name: nonamesecurity` line (and the `plugins:` line above it if nothing else is under it). Then run the commands in *Changing kong.yml later*.
2. Open `values.yaml`, delete the `plugins:` block from Step 9 (4 lines). Then:
   ```bash
   helm upgrade kong kong/kong -n kong -f values.yaml --version 3.4.1
   kubectl delete configmap kong-plugin-nonamesecurity -n kong
   ```
3. Delete the integration in the API Security UI.

Doing step 2 before step 1 makes Kong fail to start, because `kong.yml` still asks for a plugin that isn't loaded.

## Uninstall Kong completely

```bash
helm uninstall kong -n kong
kubectl delete namespace kong
```

---

## Troubleshooting

| Problem | What to do |
| --- | --- |
| `helm`: `Kubernetes cluster unreachable ... localhost:8080` | k3s: `export KUBECONFIG=/etc/rancher/k3s/k3s.yaml` (Step 1). |
| Pod stuck in `ContainerCreating` | A ConfigMap is missing. `kubectl describe pod -n kong -l app.kubernetes.io/instance=kong` names it. Redo Step 3 or Step 8. |
| Pod in `CrashLoopBackOff` | `kubectl logs deploy/kong-kong -n kong --previous`. Usually a typo in `kong.yml`. Fix it, then follow *Changing kong.yml later*. |
| Log says `nonamesecurity` plugin is not enabled / not installed | `kong.yml` uses the plugin but Step 9 wasn't done (or was undone). Do Step 9. |
| `EXTERNAL-IP` stays `<pending>` | Your cluster has no LoadBalancer, or 80/443 is already used. Set `type: NodePort` in `values.yaml`, then `helm upgrade kong kong/kong -n kong -f values.yaml --version 3.4.1`. |
| Log shows `[NONAME] Failed to connect` or `returned with an error` | Kong can't reach the engine, or the engine rejects the source key. Test from the cluster (see below). If the test returns 200, download a fresh zip and do *Updating the Noname plugin*. |
| Traffic works, but nothing in the UI and no errors | Check that `kong.yml` has `- name: nonamesecurity` under a left-edge `plugins:`. Test the engine (see below). |

**Test the engine from inside the cluster.** The Kong image has no `curl`, so use a temporary pod. Use the host from your `NN_ENGINE_URL` (Step 7). It should print `200`:

```bash
kubectl run nn-test -n kong --rm -i --restart=Never --image=curlimages/curl \
  --command -- curl -s -o /dev/null -w "%{http_code}\n" https://<your-engine-host>/engine
```

**No `[NONAME]` lines in the log is normal.** The plugin only logs when something is wrong.
