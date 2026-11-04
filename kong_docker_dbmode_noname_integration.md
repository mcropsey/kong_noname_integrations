# Kong NoName Security Plugin Install (DB Mode)

Nov 3, 2026

Kong runs in Docker in **DB mode**: the `kong` container on the stock `kong:3.9.4` image stores its config in Postgres (the `kong-db` container). The two talk over the Docker network `kongnet`. `/opt/kong/kong.yml` is the source of truth, and it's pushed to Kong with decK (`/opt/kong/kong-apply.sh`).

We build a new image, `kong-noname:3.9.4`, which is the same Kong with the NoName plugin files added. We restart Kong on the new image, and *then* turn the plugin on in `kong.yml` and push it with decK.

This is the DB-mode version of [kong_docker_dbless_noname_integration.md](kong_docker_dbless_noname_integration.md). Steps 1 through 5 (building the image) are the same in both. What's different in DB mode:

- **The order flips.** In DB-less mode you edit `kong.yml`, then restart. In DB mode you restart on the new image first, then add the plugin. Kong's Admin API rejects a plugin it hasn't loaded, so decK can't add `nonamesecurity` until Kong is running the new image.
- **No rush.** Kong running the new image with the plugin loaded but not yet turned on behaves exactly like stock Kong. You can stop between steps.
- **The plugin setting lives in Postgres**, not just in a file. That's why rollback has a set order (see [Rollback](#rollback)).
- **No database migrations.** The plugin is plain Lua files with no database tables, so `kong migrations up` isn't needed.

## Before you start

- Kong host: `192.168.1.91`, user `mcropsey`, with sudo
- Kong was installed with the DB-mode guide: containers `kong` and `kong-db`, network `kongnet`, the DB password in `~/.kong-db-password`, and `/opt/kong` owned by you, with `kong.yml` and `kong-apply.sh` in it
- Your user is in the `docker` group, so `docker ps` works without sudo. **Run every command in this guide without sudo unless it shows sudo.** The start script reads `~/.kong-db-password`, and under sudo `~` points to root's home, so the password lookup would fail.
- The plugin zip, `noname-security-kong-policy.zip`, downloaded from the API Security UI

Check that the setup matches:

```bash
docker ps --format '{{.Names}}  {{.Image}}  {{.Status}}'
curl -s localhost:8001/ | python3 -c 'import json,sys; print(json.load(sys.stdin)["configuration"]["database"])'
/opt/kong/kong-apply.sh diff
```

You should see `kong` and `kong-db` both `Up`, the database should print `postgres`, and the `diff` should end with `Created: 0  Updated: 0  Deleted: 0`. If the diff shows changes, someone changed Kong without going through `kong.yml`. Sort that out before you start, because Step 9 pushes `kong.yml` as a full sync and removes anything that isn't in the file.

## Step 1: Unzip the plugin (your computer)

```bash
unzip noname-security-kong-policy.zip -d noname-security-kong-policy
```

This unpacks the zip into a folder with the same name.

## Step 2: Copy the folder to the Kong host (your computer)

```bash
scp -r noname-security-kong-policy mcropsey@192.168.1.91:~/
```

This copies the whole folder into your home directory on the Kong host. `-r` means copy the folder and everything in it.

## Step 3: Copy the plugin files to a build folder (Kong host)

Log in to the Kong host:

```bash
ssh mcropsey@192.168.1.91
```

The plugin itself is in `~/noname-security-kong-policy/kong/plugins/nonamesecurity`. The rest of the zip isn't needed. Copy just the plugin files into the build folder. The commands use full paths, so they work from any directory:

```bash
rm -rf /opt/kong/build/nonamesecurity
mkdir -p /opt/kong/build/nonamesecurity
cp -r ~/noname-security-kong-policy/kong/plugins/nonamesecurity/. /opt/kong/build/nonamesecurity/
chmod -R a+rX /opt/kong/build/nonamesecurity
ls -la /opt/kong/build/nonamesecurity
```

The `rm -rf` clears out anything left from an earlier attempt, so only the plugin files end up in the build folder.

The `chmod` makes the files readable by everyone, so the `kong` user inside the container can read them. The `ls` must show six files: `batch_queue.lua`, `configuration.lua`, `handler.lua`, `prevention.lua`, `schema.lua`, and `utils.lua`. If the folder is empty, the `cp` failed (usually `cannot stat ... No such file or directory`), so check that Step 2 put the folder in your home directory. Don't go on to Step 4 with an empty folder.

`schema.lua` holds your integration's source key. Don't commit it to git or push the image you build to a public registry.

## Step 4: Create the Dockerfile (Kong host)

The Dockerfile tells Docker how to build the new image.

```bash
vi /opt/kong/build/Dockerfile
```

Press `i`, type these four lines, then press Esc and type `:wq` to save:

```
FROM kong:3.9.4
USER root
COPY nonamesecurity /usr/local/share/lua/5.1/kong/plugins/nonamesecurity
USER kong
```

What each line does:

- `FROM kong:3.9.4` starts from the stock Kong image. It must match the version your `kong` container runs now.
- `USER root` switches to root so the files can be copied in.
- `COPY ...` copies the plugin folder to where Kong looks for plugins.
- `USER kong` switches back to the normal Kong user.

## Step 5: Build the new image (Kong host)

```bash
cd /opt/kong/build
docker build -t kong-noname:3.9.4 .
```

`-t kong-noname:3.9.4` names the new image. The `.` at the end means build from the current folder.

Check that the plugin made it into the image:

```bash
docker run --rm kong-noname:3.9.4 ls /usr/local/share/lua/5.1/kong/plugins/nonamesecurity
```

This starts a throwaway container, lists the plugin folder, then deletes the container. You should see the same six `.lua` files from Step 3. Your running Kong isn't touched.

## Step 6: Create the start script (Kong host)

First save a copy of how the current Kong container was started, in case you need to compare or roll back:

```bash
docker inspect kong > ~/kong-before.json
chmod 600 ~/kong-before.json
```

The `chmod` matters: in DB mode the `inspect` output includes the database password, so keep the file private.

Check whether a start script already exists. **On a host set up with the DB-mode install guide (like .91), it won't exist yet, and that's normal.** That guide starts Kong with a plain `docker run` command and never creates a start script.

```bash
ls -la /opt/kong/run-kong.sh
```

- **If you see `ls: cannot access '/opt/kong/run-kong.sh': No such file or directory`,** that's the normal case. Nothing is wrong and there's nothing to back up. vi creates the file in the next step.
- **If the file exists,** back it up first with `cp /opt/kong/run-kong.sh /opt/kong/run-kong.sh.bak`. When you open it in vi, type `:%d` and press Enter to clear the old contents before you paste the new ones.

Create the script:

```bash
vi /opt/kong/run-kong.sh
```

Type `:set paste` and press Enter (this stops vi from adding extra spaces when you paste). Press `i`, paste the lines below, press Esc, then `:wq`:

```bash
#!/bin/bash
# Start Kong (DB mode) on the image given as $1 (default: kong-noname:3.9.4).
# Run as your normal user, not with sudo: the password is read from ~/.kong-db-password.
set -euo pipefail

IMAGE=${1:-kong-noname:3.9.4}

# Use KONG_PLUGINS=bundled,nonamesecurity only on an image that has the plugin
if [ "$IMAGE" = "kong:3.9.4" ]; then
  PLUGINS=bundled
else
  PLUGINS=bundled,nonamesecurity
fi

PW_FILE="$HOME/.kong-db-password"
[ -r "$PW_FILE" ] || { echo "can't read $PW_FILE (did you run this with sudo?)"; exit 1; }

# Remove the old Kong container (the kong-db container and its data are not touched)
docker rm -f kong

# Start Kong in DB mode on the chosen image
docker run -d --name kong \
  --restart unless-stopped \
  --network kongnet \
  -p 80:8000 \
  -p 8000:8000 \
  -p 8001:8001 \
  -e KONG_DATABASE=postgres \
  -e KONG_PG_HOST=kong-db \
  -e KONG_PG_USER=kong \
  -e KONG_PG_PASSWORD="$(cat "$PW_FILE")" \
  -e KONG_ADMIN_LISTEN=0.0.0.0:8001 \
  -e KONG_PROXY_ACCESS_LOG=/dev/stdout \
  -e KONG_ADMIN_ACCESS_LOG=/dev/stdout \
  -e KONG_PROXY_ERROR_LOG=/dev/stderr \
  -e KONG_ADMIN_ERROR_LOG=/dev/stderr \
  -e KONG_LOG_LEVEL=notice \
  -e KONG_NGINX_PROXY_REAL_IP_HEADER=X-Real-IP \
  -e KONG_PLUGINS="$PLUGINS" \
  "$IMAGE"

echo "started kong on $IMAGE with KONG_PLUGINS=$PLUGINS"
```

Make it runnable:

```bash
chmod +x /opt/kong/run-kong.sh
```

What the main options do:

| Option | What it does |
| --- | --- |
| `--restart unless-stopped` | Docker starts Kong again after a crash or reboot |
| `--network kongnet` | Lets Kong reach Postgres by its name, `kong-db`. Without it, Kong can't find its database and won't start. |
| `-p 80:8000`, `-p 8000:8000` | Traffic to port 80 or 8000 on the host goes to Kong's proxy |
| `-p 8001:8001` | Kong's Admin API, used by decK and the checks in Steps 7 and 10 |
| `KONG_DATABASE=postgres` | DB mode: Kong reads its config from Postgres |
| `KONG_PG_HOST / USER / PASSWORD` | How Kong connects to Postgres. The password is read from `~/.kong-db-password`, so it isn't written in the script. |
| `KONG_ADMIN_LISTEN=0.0.0.0:8001` | Since Kong 3.9 the Admin API listens only inside the container unless you set this |
| `KONG_*_LOG` | Sends Kong's logs to `docker logs` (access logs to stdout, errors to stderr) |
| `KONG_PLUGINS=bundled,nonamesecurity` | Turns on Kong's built-in plugins plus NoName. Keep `bundled`, or plugins like rate-limiting stop working. |
| `kong-noname:3.9.4` | The new image you built in Step 5 (the default when no image is given) |

There's no `kong.yml` mount and no `KONG_DECLARATIVE_CONFIG` here. Those are for DB-less mode. In DB mode, Kong gets its config from Postgres, and decK keeps Postgres matching `kong.yml`.

This script matches the original container on this host. To double-check, print the old container's settings from `~/kong-before.json` (the password line is left out on purpose):

```bash
python3 -c 'import json; c=json.load(open("/home/mcropsey/kong-before.json"))[0]; print("\n".join(e for e in c["Config"]["Env"] if e.startswith("KONG_") and "PASSWORD" not in e)); print(c["HostConfig"]["PortBindings"]); print(c["HostConfig"]["NetworkMode"])'
```

The `KONG_*` settings should match the script's `-e` lines, except `KONG_PLUGINS`, which is new. The ports should be 80, 8000, and 8001, and the network should be `kongnet`. If your old container had other ports or `-e` settings, add them to the script.

## Step 7: Restart Kong on the new image (Kong host)

```bash
/opt/kong/run-kong.sh
```

This removes the old container and starts the new one. Kong is down for a few seconds. Your services and routes are all still in Postgres, so they come back on their own.

Wait for Kong to report `healthy`:

```bash
until [ "$(docker inspect kong --format '{{.State.Health.Status}}')" = healthy ]; do sleep 2; done; echo kong is healthy
```

Then check that Kong loaded the plugin code:

```bash
curl -s localhost:8001/plugins/enabled
```

The list should include `nonamesecurity`. At this point the plugin is **loaded but not turned on**. Traffic goes through Kong exactly as before. If you need to stop here, you can.

## Step 8: Add the plugin to kong.yml (Kong host)

Back up the file first:

```bash
cp /opt/kong/kong.yml /opt/kong/kong.yml.bak
```

Open the file in vi:

```bash
vi /opt/kong/kong.yml
```

Turn off auto-indent first: type `:set paste` and press Enter. Without this, vi may indent a new line on its own when you press `o`, so the cursor doesn't start at the left edge and your spaces get added on top of vi's.

YAML rules to keep in mind while typing:

- Use the spacebar for indentation, never Tab. YAML doesn't allow tabs.
- `plugins:` starts at the left edge, with no spaces before it. This makes the plugin global, so it applies to every service.
- `- name: nonamesecurity` is indented exactly two spaces, so the dash lines up with the dashes of any other plugin entries.
- There is a space after the dash and a space after the colon.

Search for the plugins section: type `/^plugins:` and press Enter.

**If vi finds it:** the cursor jumps to the `plugins:` line. Press `o` to open a new line below it. Type two spaces, then `- name: nonamesecurity`. Press Esc, then type `:wq` to save. Leave the existing plugins as they are. The result looks like this:

```yaml
plugins:
  - name: nonamesecurity
  - name: rate-limiting
    config:
      ...
```

**If vi says `Pattern not found`** (the case on a fresh DB-mode install, which has no plugins): the file has no plugins section yet. Press `G` to jump to the last line, then `o` to open a new line. Type `plugins:` and press Enter. On the next line, type two spaces, then `- name: nonamesecurity`. Press Esc, then type `:wq` to save. The end of the file looks like this:

```yaml
plugins:
  - name: nonamesecurity
```

If a line came out wrong, open the file again, move to that line, press `0` to jump to the start of the line, then `x` to delete an extra space, or `i` and the spacebar to add one. Step 9 checks the file before anything changes, so a mistake here gets caught.

If you'd rather turn NoName on for only one service, put the `plugins:` block under that service instead (indented to line up with the service's `routes:`), not at the left edge.

## Step 9: Check and push kong.yml (Kong host)

Check the file against the running Kong:

```bash
/opt/kong/kong-apply.sh validate
```

This must succeed. It only works now that Kong is running the new image. On the old image, it fails because Kong doesn't know `nonamesecurity`. That's the reason for the order of Steps 7 and 8.

Preview the change:

```bash
/opt/kong/kong-apply.sh diff
```

It should show one plugin, `nonamesecurity`, being created, and nothing else: `Created: 1  Updated: 0  Deleted: 0`. **If it shows anything else being deleted or updated, stop** and find out why before you apply.

Push it:

```bash
/opt/kong/kong-apply.sh apply
```

This saves a backup of the current Kong state to `~/kong-backups/`, then pushes `kong.yml`. No restart is needed: Kong picks up the change within about 5 seconds.

## Step 10: Check that it works (Kong host)

```bash
docker ps
```

The `kong` line should show image `kong-noname:3.9.4` and status `Up ... (healthy)`. `kong-db` should still be `Up`.

```bash
curl -s localhost:8001/plugins | python3 -c 'import json,sys; [print(p["name"], p["enabled"]) for p in json.load(sys.stdin)["data"]]'
```

The list should include `nonamesecurity True`.

```bash
/opt/kong/kong-apply.sh diff
```

It should show no changes, meaning Kong matches `kong.yml`.

```bash
docker logs kong 2>&1 | grep NONAME
```

No output is good: the plugin only logs when something goes wrong. The `2>&1` matters, because Kong writes errors to stderr and a plain `| grep` never sees them.

Finally, send some traffic through Kong and check that it shows up in the API Security UI. The first API can take a few minutes to appear.

## Step 11: Make sure Kong survives a reboot (Kong host)

```bash
sudo systemctl enable docker
sudo reboot
```

The first command makes Docker start at boot. Once the host is back up, log in and run:

```bash
docker ps
```

`kong` and `kong-db` should both be `Up` without you starting them, and `curl -s localhost:8001/plugins/enabled` should still list `nonamesecurity`.

## Updating to a new plugin version

The plugin setting is stored in Postgres and stays there, so an update is just new code:

1. Download the new zip from the API Security UI (the `schema.lua` is filled in for your integration again), then redo Steps 1 through 3.
2. Build with a new tag so you can go back to the old one: `docker build -t kong-noname:3.9.4-<new> .`
3. Restart on it: `/opt/kong/run-kong.sh kong-noname:3.9.4-<new>`
4. Run `/opt/kong/kong-apply.sh diff`. Normally it shows no changes. If the release notes say a setting you put in `kong.yml` was renamed or removed, fix `kong.yml` and `apply`.

You don't need to change `kong.yml` or re-add the plugin. To go back, run `/opt/kong/run-kong.sh` with the old tag.

## Rollback

**Do these in this order.** In DB mode the plugin setting is stored in Postgres. If you switch back to stock Kong while Postgres still has `nonamesecurity` turned on, stock Kong fails at startup with `nonamesecurity plugin is in use but not enabled`. So remove the plugin from Kong first, then change the image.

1. Restore `kong.yml` and push it, which removes the plugin from Postgres:

```bash
cp /opt/kong/kong.yml.bak /opt/kong/kong.yml
/opt/kong/kong-apply.sh diff
/opt/kong/kong-apply.sh apply
```

The `diff` should show only `nonamesecurity` being deleted: `Deleted: 1`.

2. Check that the plugin is gone:

```bash
curl -s localhost:8001/plugins | grep -c nonamesecurity
```

This should print `0`.

3. Restart on stock Kong:

```bash
/opt/kong/run-kong.sh kong:3.9.4
```

The script sets `KONG_PLUGINS=bundled` on its own for the stock image.

If the plugin needs to be off right now, step 1 alone does it: within about 5 seconds the plugin stops handling traffic, while Kong keeps running the `kong-noname` image. You can do steps 2 and 3 later.

## Troubleshooting

| Problem | What to check |
| --- | --- |
| `docker build` says `COPY failed`, or the Step 5 `ls` is empty | `/opt/kong/build/nonamesecurity` is empty. Redo Step 3 and confirm its `ls` shows six files. |
| `pull access denied for kong-noname` | The image was never built. Redo Step 5 and read the error. |
| `can't read /root/.kong-db-password` or a Postgres `password authentication failed` | You ran `run-kong.sh` with sudo. Run it as your normal user. |
| Kong keeps restarting after Step 7, and `docker logs kong` shows it can't connect to `kong-db` | The container isn't on `kongnet`, or `kong-db` isn't running. Check `docker ps` and the `--network kongnet` line. |
| `/plugins/enabled` doesn't list `nonamesecurity` after Step 7 | `KONG_PLUGINS` is missing `nonamesecurity`, or Kong is still on `kong:3.9.4`. Check `docker ps` for the image name. |
| Step 9 `validate` says `plugin 'nonamesecurity' not enabled` or `no configuration schema found` | Kong isn't running the new image with the plugin loaded. Redo Step 7. If the image is right, the files are nested too deep: redo Step 3 (including the `rm -rf`), then rebuild with `docker build --no-cache -t kong-noname:3.9.4 .` |
| Step 9 shows a YAML error | Indentation in `kong.yml` is off. See Step 8. |
| Step 9 `diff` wants to delete things you didn't touch | Someone changed Kong without going through `kong.yml`. Don't apply. Run `/opt/kong/kong-apply.sh backup`, compare the backup with `kong.yml`, and add what's missing to `kong.yml`. |
| Stock Kong won't start after a rollback, and `docker logs kong` shows `nonamesecurity plugin is in use but not enabled` | You switched images before removing the plugin. Run `/opt/kong/run-kong.sh` (back on `kong-noname`), then do the Rollback steps in order. |
| Rate limiting stopped working | `bundled` is missing from `KONG_PLUGINS` in the script. |
| `docker logs` shows `[NONAME]` errors | Kong can't reach the NoName engine, or the zip doesn't match your integration. Download the zip again. |
| Nothing shows up in the UI | Check that `curl -s localhost:8001/plugins` lists `nonamesecurity`, and that your traffic actually goes through this Kong. |
