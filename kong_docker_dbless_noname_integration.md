# Kong NoName Security Plugin Install

Nov 3, 2026

Kong runs in Docker on the stock `kong:3.9.4` image. We build a new image, `kong-noname:3.9.4`, which is the same Kong with the NoName plugin files added. Then we turn the plugin on in `kong.yml` and restart Kong on the new image.

## Before you start

- Kong host: `192.168.1.91`, user `mcropsey`, with sudo
- The plugin zip, `noname-security-kong-policy.zip`, downloaded from the API Security UI
- Do Steps 6 through 9 in one sitting. Once `kong.yml` lists the plugin, the old Kong container can't load it, so don't leave Kong half done.

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
sudo rm -rf /opt/kong/build/nonamesecurity
sudo mkdir -p /opt/kong/build/nonamesecurity
sudo cp -r ~/noname-security-kong-policy/kong/plugins/nonamesecurity/. /opt/kong/build/nonamesecurity/
sudo chmod -R a+rX /opt/kong/build/nonamesecurity
ls -la /opt/kong/build/nonamesecurity
```

The `rm -rf` clears out anything left from an earlier attempt, so only the plugin files end up in the build folder.

The `chmod` makes the files readable by everyone, so the `kong` user inside the container can read them. The `ls` must show six files: `batch_queue.lua`, `configuration.lua`, `handler.lua`, `prevention.lua`, `schema.lua`, and `utils.lua`. If the folder is empty, the `cp` failed (usually `cannot stat ... No such file or directory`), so check that Step 2 put the folder in your home directory. Don't go on to Step 4 with an empty folder.

## Step 4: Create the Dockerfile (Kong host)

The Dockerfile tells Docker how to build the new image.

```bash
sudo vi /opt/kong/build/Dockerfile
```

Press `i`, type these four lines, then press Esc and type `:wq` to save:

```
FROM kong:3.9.4
USER root
COPY nonamesecurity /usr/local/share/lua/5.1/kong/plugins/nonamesecurity
USER kong
```

What each line does:

- `FROM kong:3.9.4` starts from the stock Kong image.
- `USER root` switches to root so the files can be copied in.
- `COPY ...` copies the plugin folder to where Kong looks for plugins.
- `USER kong` switches back to the normal Kong user.

## Step 5: Build the new image (Kong host)

```bash
cd /opt/kong/build
sudo docker build -t kong-noname:3.9.4 .
```

`-t kong-noname:3.9.4` names the new image. The `.` at the end means build from the current folder.

Check that the plugin made it into the image:

```bash
sudo docker run --rm kong-noname:3.9.4 ls /usr/local/share/lua/5.1/kong/plugins/nonamesecurity
```

This starts a throwaway container, lists the plugin folder, then deletes the container. You should see the same six `.lua` files from Step 3. Your running Kong isn't touched.

## Step 6: Add the plugin to kong.yml (Kong host)

Back up the file first:

```bash
sudo cp /opt/kong/kong.yml /opt/kong/kong.yml.bak
```

Open the file in vi:

```bash
sudo vi /opt/kong/kong.yml
```

Turn off auto-indent first: type `:set paste` and press Enter. Without this, vi may indent a new line on its own when you press `o`, so the cursor doesn't start at the left edge and your spaces get added on top of vi's.

YAML rules to keep in mind while typing:

- Use the spacebar for indentation, never Tab. YAML doesn't allow tabs.
- `plugins:` starts at the left edge, with no spaces before it.
- `- name: nonamesecurity` is indented exactly two spaces, so the dash lines up with the dashes of any other plugin entries.
- There is a space after the dash and a space after the colon.

Search for the plugins section: type `/^plugins:` and press Enter.

**If vi finds it (most customer installs):** the cursor jumps to the `plugins:` line. Press `o` to open a new line below it. Type two spaces, then `- name: nonamesecurity`. Press Esc, then type `:wq` to save. Leave the existing plugins as they are. The result looks like this:

```yaml
plugins:
  - name: nonamesecurity
  - name: rate-limiting
    config:
      ...
```

**If vi says `Pattern not found`:** the file has no plugins section yet. Press `G` to jump to the last line, then `o` to open a new line. Type `plugins:` and press Enter. On the next line, type two spaces, then `- name: nonamesecurity`. Press Esc, then type `:wq` to save. The end of the file looks like this:

```yaml
plugins:
  - name: nonamesecurity
```

If a line came out wrong, open the file again, move to that line, press `0` to jump to the start of the line, then `x` to delete an extra space, or `i` and the spacebar to add one. Step 7 checks the file before anything restarts, so a mistake here gets caught.

Make sure Kong can read the file:

```bash
sudo chmod a+r /opt/kong/kong.yml
```

## Step 7: Check kong.yml (Kong host)

This starts a throwaway Kong from the new image and asks it to read `kong.yml`. It catches mistakes before you restart the real Kong.

```bash
sudo docker run --rm -v /opt/kong/kong.yml:/k.yml:ro -e KONG_DATABASE=off -e KONG_PLUGINS=bundled,nonamesecurity kong-noname:3.9.4 kong config parse /k.yml
```

It should end with `parse successful`. A `ulimit` warning above it is normal. If you get an error, fix `kong.yml` and run the check again.

## Step 8: Create the start script (Kong host)

First save a copy of how the current Kong container was started, in case you need to compare or roll back:

```bash
sudo docker inspect kong > ~/kong-before.json
```

Check whether a start script already exists:

```bash
ls -la /opt/kong/run-kong.sh
```

- **`No such file or directory`** is normal. It means Kong was started by hand with a `docker run` command. vi creates the file in the next step. Use `~/kong-before.json` to make sure the new script has the same ports and settings as the old container.
- **If the file exists,** back it up first with `sudo cp /opt/kong/run-kong.sh /opt/kong/run-kong.sh.bak`. When you open it in vi, type `:%d` and press Enter to clear the old contents before you paste the new ones.

Create the script:

```bash
sudo vi /opt/kong/run-kong.sh
```

Type `:set paste` and press Enter (this stops vi from adding extra spaces when you paste). Press `i`, paste the lines below, press Esc, then `:wq`:

```bash
#!/bin/bash

# Remove the old Kong container
docker rm -f kong

# Start Kong on the new image with the NoName plugin
docker run -d --name kong \
  --restart unless-stopped \
  -p 80:8000 \
  -p 8000:8000 \
  -p 8001:8001 \
  -v /opt/kong/kong.yml:/etc/kong/kong.yml:ro \
  -e KONG_DATABASE=off \
  -e KONG_DECLARATIVE_CONFIG=/etc/kong/kong.yml \
  -e KONG_PROXY_LISTEN=0.0.0.0:8000 \
  -e KONG_ADMIN_LISTEN=0.0.0.0:8001 \
  -e KONG_PROXY_ACCESS_LOG=/dev/stdout \
  -e KONG_ADMIN_ACCESS_LOG=/dev/stdout \
  -e KONG_PROXY_ERROR_LOG=/dev/stderr \
  -e KONG_ADMIN_ERROR_LOG=/dev/stderr \
  -e KONG_LOG_LEVEL=notice \
  -e KONG_NGINX_PROXY_REAL_IP_HEADER=X-Real-IP \
  -e KONG_PLUGINS=bundled,nonamesecurity \
  kong-noname:3.9.4
```

Make it runnable:

```bash
sudo chmod +x /opt/kong/run-kong.sh
```

What the main options do:

| Option | What it does |
| --- | --- |
| `--restart unless-stopped` | Docker starts Kong again after a crash or reboot |
| `-p 80:8000`, `-p 8000:8000` | Traffic to port 80 or 8000 on the host goes to Kong's proxy |
| `-p 8001:8001` | Kong's Admin API, used for the checks in Step 10 |
| `-v /opt/kong/kong.yml:...` | Gives the container your `kong.yml` (read-only) |
| `KONG_*_LOG` | Sends Kong's logs to `docker logs` (access logs to stdout, errors to stderr) |
| `KONG_DATABASE=off` | Kong reads its config from `kong.yml`, not a database |
| `KONG_PLUGINS=bundled,nonamesecurity` | Turns on Kong's built-in plugins plus NoName. Keep `bundled`, or plugins like rate-limiting stop working. |
| `kong-noname:3.9.4` | The new image you built in Step 5 |

This script matches the original container on this host. If your old container used other ports or `-e` settings, look for them in `~/kong-before.json` (under `Env` and `PortBindings`) and add them to the script.

## Step 9: Restart Kong on the new image (Kong host)

```bash
sudo /opt/kong/run-kong.sh
```

This removes the old container and starts the new one. Kong is down for a few seconds.

## Step 10: Check that it works (Kong host)

Wait about 10 seconds, then:

```bash
sudo docker ps
```

The `kong` line should show image `kong-noname:3.9.4` and status `Up`.

```bash
curl -s localhost:8001/plugins/enabled
```

The list should include `nonamesecurity`.

```bash
sudo docker logs kong 2>&1 | grep NONAME
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
sudo docker ps
```

Kong should be `Up` without you starting it.

## Rollback

To go back to stock Kong without the plugin:

```bash
sudo cp /opt/kong/kong.yml.bak /opt/kong/kong.yml
sudo chmod a+r /opt/kong/kong.yml
sudo vi /opt/kong/run-kong.sh
```

In the script, change `kong-noname:3.9.4` to `kong:3.9.4`, and `KONG_PLUGINS=bundled,nonamesecurity` to `KONG_PLUGINS=bundled`. Save, then run:

```bash
sudo /opt/kong/run-kong.sh
```

## Troubleshooting

| Problem | What to check |
| --- | --- |
| `docker build` says `COPY failed`, or the Step 5 `ls` is empty | `/opt/kong/build/nonamesecurity` is empty. Redo Step 3 and confirm its `ls` shows six files. |
| `pull access denied for kong-noname` | The image was never built. Redo Step 5 and read the error. |
| Step 7 says `no configuration schema found for plugin: nonamesecurity` | The plugin files are nested too deep in the image. Redo Step 3 (including the `rm -rf`), then rebuild with `sudo docker build --no-cache -t kong-noname:3.9.4 .` |
| Step 7 shows a YAML error | Indentation in `kong.yml` is off. See Step 6. |
| Kong keeps restarting | Run `sudo docker logs kong`. Usually a `kong.yml` mistake, or the file isn't readable (`sudo chmod a+r /opt/kong/kong.yml`). |
| Rate limiting stopped working | `bundled` is missing from `KONG_PLUGINS` in the script. |
| `docker logs` shows `[NONAME]` errors | Kong can't reach the NoName engine, or the zip doesn't match your integration. Download the zip again. |
| Nothing shows up in the UI | Check that `curl -s localhost:8001/plugins` lists `nonamesecurity`, and that your traffic actually goes through this Kong. |
