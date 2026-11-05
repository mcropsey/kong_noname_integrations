# Noname Plugin Integration: Kong on Kubernetes (Helm, DB-less)

This guide adds the Noname (API Security) plugin to a Kong gateway that is already running on Kubernetes. It follows Noname's *Kong Plugin - Kubernetes Helm* doc.

Tested on k3s v1.37 with Kong 3.9.4, Helm chart `kong/kong` 3.4.1 and Noname plugin 5.0.1.

---

## Before you start

Your Kong must already be:

- installed with the Kong Helm chart, release `kong`, namespace `kong` (Deployment `kong-kong`)
- in DB-less mode, with its config in `kong.yml`, loaded from ConfigMap `kong-dbless-config` (`dblessConfig.configMap` in your values file)
- working: traffic through Kong returns `200`

You also need:

- the values file your Kong was installed with (this guide calls it `values.yaml`) and your `kong.yml`, in one folder (this guide uses `~/kong-k8s`)
- `helm` and `kubectl` working. **On k3s**, run `export KUBECONFIG=/etc/rancher/k3s/k3s.yaml` first.
- the cluster can reach your API Security engine on the internet (HTTPS), and you can log in to the API Security UI

Kong keeps working after every step.

---

## Step 1: Download the plugin

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

## Step 2: Load the plugin into the cluster

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

## Step 3: Enable the plugin in Kong

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

The plugin is loaded but not yet looking at traffic. Step 4 turns it on.

## Step 4: Turn the plugin on for all traffic

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

## Step 5: Check Noname works

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

**2. Traffic still goes through Kong.** Use your Kong proxy address (`kubectl get svc kong-kong-proxy -n kong`) and one of your hosts. This should print `200`:

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

1. Download a new zip from the UI (Step 1). Unzip it into a **new, empty** `noname-security-kong-policy` folder. Don't mix files from two downloads.
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
2. Open `values.yaml`, delete the `plugins:` block from Step 3 (4 lines). Then:
   ```bash
   helm upgrade kong kong/kong -n kong -f values.yaml --version 3.4.1
   kubectl delete configmap kong-plugin-nonamesecurity -n kong
   ```
3. Delete the integration in the API Security UI.

Doing step 2 before step 1 makes Kong fail to start, because `kong.yml` still asks for a plugin that isn't loaded.

## Troubleshooting

| Problem | What to do |
| --- | --- |
| `helm`: `Kubernetes cluster unreachable ... localhost:8080` | k3s: `export KUBECONFIG=/etc/rancher/k3s/k3s.yaml`. |
| Pod stuck in `ContainerCreating` | A ConfigMap is missing. `kubectl describe pod -n kong -l app.kubernetes.io/instance=kong` names it. Redo Step 2. |
| Pod in `CrashLoopBackOff` | `kubectl logs deploy/kong-kong -n kong --previous`. Usually a typo in `kong.yml`. Fix it, then follow *Changing kong.yml later*. |
| Log says `nonamesecurity` plugin is not enabled / not installed | `kong.yml` uses the plugin but Step 3 wasn't done (or was undone). Do Step 3. |
| Log shows `[NONAME] Failed to connect` or `returned with an error` | Kong can't reach the engine, or the engine rejects the source key. Test from the cluster (see below). If the test returns 200, download a fresh zip and do *Updating the Noname plugin*. |
| Traffic works, but nothing in the UI and no errors | Check that `kong.yml` has `- name: nonamesecurity` under a left-edge `plugins:`. Test the engine (see below). |

**Test the engine from inside the cluster.** The Kong image has no `curl`, so use a temporary pod. Use the host from your `NN_ENGINE_URL` (Step 1). It should print `200`:

```bash
kubectl run nn-test -n kong --rm -i --restart=Never --image=curlimages/curl \
  --command -- curl -s -o /dev/null -w "%{http_code}\n" https://<your-engine-host>/engine
```

**No `[NONAME]` lines in the log is normal.** The plugin only logs when something is wrong.
