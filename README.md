# ComeOver for UmbrelOS

This directory is a complete **Umbrel community app store** containing the ComeOver relay.

```
deploy/umbrel/
├── umbrel-app-store.yml       # store id + display name
└── comeover/
    ├── umbrel-app.yml         # app listing, port, icon
    ├── docker-compose.yml     # app_proxy + relay
    └── icon.svg               # source for the embedded manifest icon
```

The app opens a small relay dashboard on port **8788** behind Umbrel auth. Phones, tablets and TVs
connect directly to port **8787** for REST and Socket.IO, because companion clients cannot send
Umbrel's login cookies. The same relay also serves `/j/<code>` invite pages and `/healthz`.

`umbrel-app.yml` embeds the app icon as a data URI, so the listing renders even before the store
repository has a public asset URL. `icon.svg` is the source if you want to host it yourself later.

## 1. Publish the relay image

Umbrel app packages must pull a prebuilt image; they do not build from source during install. The
included workflow at `.github/workflows/relay-image.yml` builds `linux/amd64` and `linux/arm64` and
pushes to GitHub Container Registry.

1. Push this repository to GitHub.
2. Run **Actions → Relay image → Run workflow** (or push a `v0.1.0` tag).
3. Open the package in GitHub → **Packages** → `comeover-relay` → **Package settings** and make it
   **public**. Umbrel cannot pull a private image.
4. Copy the digest printed by the workflow and pin it in `comeover/docker-compose.yml`:

   ```yaml
   image: ghcr.io/<owner>/comeover-relay:0.1.0@sha256:<digest>
   ```

5. If your GitHub owner is not `github-user0008`, update the image name in
   `comeover/docker-compose.yml`. Keep the tag in step with `version:` in `umbrel-app.yml`.

To build and push manually instead:

```bash
cd server
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag ghcr.io/<owner>/comeover-relay:0.1.0 \
  --push .
```

### Offline / local-image alternative

If you do not want a registry, build the image directly on the Umbrel box with the same tag used
in the compose file:

```bash
cd comeover/server
docker build -t ghcr.io/github-user0008/comeover-relay:0.1.0 .
```

Compose will use the local image when the tag already exists. This is fine for a private box, but
you will need to rebuild it after every relay update.

## 2. Make the store available to Umbrel

Umbrel clones the **root** of a community app-store repository, so this `deploy/umbrel` directory
must be the root of its own repository (or copied into one). A separate store repo keeps the
Flutter client out of Umbrel's clone:

```bash
cd deploy/umbrel
git init -b main
git add .
git commit -m "ComeOver Umbrel app store"
git remote add origin git@github.com:<owner>/comeover-umbrel.git
git push -u origin main
```

You can also point Umbrel at this whole repository only if you move `umbrel-app-store.yml` and the
`comeover/` app folder to its root.

In umbrelOS:

1. Open **App Store**.
2. Open the **⋯** menu → **Community App Stores**.
3. Paste the store repository URL and add it.
4. Open **ComeOver App Store** and install **ComeOver**.

The app should install on the default `8788` port. Open it from the Umbrel home screen to see the
relay dashboard.

### If port 8787 or 8788 is already taken

The manifest port and the direct relay port are intentionally separate:

| Port | Purpose | Auth |
| --- | --- | --- |
| `8788` | Umbrel dashboard (`app_proxy`) | Umbrel login |
| `8787` | REST + Socket.IO + invite pages | Room tokens only |
| `8240` | Tailscale app UI (if installed) | Tailscale |

Change both `port:` in `umbrel-app.yml` and the host side of `"8787:8787"` if they conflict. Also
change `COMEOVER_DIRECT_PORT` so the dashboard prints the correct address.

## 3. Put the Umbrel on Tailscale

Install the **Tailscale** app from the Umbrel App Store and sign in. The official Umbrel package
runs Tailscale in host-network mode, so services published by Umbrel apps are reachable on the
tailnet without extra configuration.

Find the Umbrel's tailnet address in the Tailscale app UI (port `8240`). If you prefer a shell,
the container is usually named `tailscale_web_1`:

```bash
sudo docker exec tailscale_web_1 tailscale status
sudo docker exec tailscale_web_1 tailscale ip -4
```

Use the `100.x.y.z` address or the MagicDNS name in the ComeOver client:

```
Settings → Relay → http://100.x.y.z:8787
```

The relay dashboard at `http://100.x.y.z:8788` also prints the direct address it derived from the
hostname you used to open it.

### Optional HTTPS with Tailscale Serve

Browsers only expose geolocation on HTTPS or localhost. The native Android/iOS apps allow the
self-hosted HTTP relay, but the web client needs HTTPS. Tailscale Serve gives the relay a trusted
certificate on your tailnet:

```bash
sudo docker exec tailscale_web_1 tailscale serve --bg http://127.0.0.1:8787
sudo docker exec tailscale_web_1 tailscale serve status
```

Then use `https://umbrel.<your-tailnet>.ts.net` as both `COMEOVER_SERVER` and
`COMEOVER_PUBLIC_URL`. Serve terminates TLS in the Tailscale container and forwards to the relay
on the host loopback interface.

Funnel is the public version and does **not** require the guest to install Tailscale:

```bash
sudo docker exec tailscale_web_1 tailscale funnel --bg http://127.0.0.1:8787
```

Funnel exposes the relay to the public internet. The relay has no accounts beyond room codes, so
use it only temporarily and turn it off when you are done:

```bash
sudo docker exec tailscale_web_1 tailscale funnel --bg off
```

## 4. Let a guest in

The guest must be able to reach the Umbrel over the network. There are three sensible options.

### Share the Umbrel node (recommended)

In the Tailscale admin console: **Machines → Umbrel → Share**. The guest installs Tailscale,
accepts the share, and can reach only that machine. They then enter the relay address from the
dashboard in **Settings → Relay**.

This is better than adding a guest to your whole tailnet. If you do invite them into the tailnet,
use a tag and an ACL that only allows TCP `8787` (and `8788` if they need the dashboard) to the
Umbrel.

### Same LAN

If the guest is on your Wi-Fi, use the Umbrel's LAN address:

```
Settings → Relay → http://192.168.1.x:8787
```

No Tailscale client is needed in that case.

### Public Funnel (temporary)

Use only for a one-off demo. Turn Funnel off when the room ends.

## 5. TVs

Fire TV and most Android TV boxes do not have a Tailscale client. Keep the TV on the home LAN and
point **Settings → Relay** at the Umbrel's LAN address:

```
http://192.168.1.x:8787
```

The TV and the Umbrel are on the same network, so the TV can reach the relay even though it is not
on the tailnet. The host phone uses the tailnet URL so the invite links it shares work for a guest
who is away from home. Both devices still meet in the same room on the relay.

If the TV is on a different VLAN or a guest network that cannot reach the Umbrel, fix that routing
first; Tailscale on the TV is not an option on most Fire OS devices.

## Tailscale subnet routers, exit nodes, and why ComeOver does not need them

A **subnet router** is a Tailscale node that advertises a LAN CIDR it can reach, for example
`192.168.1.0/24`. Once an admin approves the advertised route, other devices on the tailnet can
send traffic for `192.168.1.x` to that node, which forwards it onto the LAN. It is how you reach
devices that cannot run Tailscale themselves, such as printers, NAS boxes, IoT gear, or a TV.

The setup is:

1. On the router-capable machine, enable IP forwarding and allow forwarding in the firewall.
2. Advertise the route:

   ```bash
   tailscale up --advertise-routes=192.168.1.0/24
   ```

   In the Umbrel Tailscale app, run that inside `tailscale_web_1`, or use the app UI if it exposes
   route advertisement.

3. Approve the route in the Tailscale admin console: **Machines → the node → Edit route settings**.
4. On clients that should use it, accept routes:

   ```bash
   tailscale up --accept-routes
   ```

   On mobile clients this is usually **Settings → Use Tailscale subnets**.

An **exit node** is a different feature: it routes a client's *default* internet traffic through
the selected node. A subnet router only adds routes for specific private CIDRs. You do not want an
exit node just to reach a home TV.

On Umbrel, the Tailscale app runs in host-network mode and deliberately disables `accept-routes` on
startup. Accepting a route that overlaps the Umbrel's own LAN can make the Umbrel unreachable on its
LAN address. If you enable it, keep the advertised prefix specific and make sure it does not overlap
the network the Umbrel itself uses.

**ComeOver does not need subnet routing.** The relay runs on the Umbrel itself, which is already a
tailnet node, so a phone connects directly to `http://<umbrel-tailnet-ip>:8787`. A TV on the home
LAN connects directly to the Umbrel's LAN IP. Subnet routing would help only if you wanted a
tailnet device to reach some *other* LAN-only device; it does not make the TV a tailnet client and
it does not change how the TV reaches the relay.

The two cases where you might still use it:

- You want to reach the TV from outside the house for sideloading, remote control, or diagnostics.
- You run the relay on a different machine on the LAN instead of on the Umbrel. In that case the
  Umbrel could advertise the LAN route so tailnet phones can reach the relay's LAN IP. Running the
  relay on the Umbrel is simpler.

## Updating

1. Bump `version:` in `umbrel-app.yml`, the image tag in `docker-compose.yml`, and
   `releaseNotes:`.
2. Publish the new image.
3. Push the store repo. Umbrel checks community stores periodically; use the App Store's update
   action when the new version appears.
4. Active rooms are lost on restart because the relay stores them in memory.

## Troubleshooting

- **The app opens but the client cannot connect.** Use port `8787`, not the dashboard port `8788`.
  The app proxy whitelists `/api`, `/socket.io*`, `/j` and `/healthz`, but the direct port is the
  least surprising path for native clients.
- **401 or Umbrel login page in the client.** The client is pointed at `8788`. Switch it to
  `8787`, or use Serve/Funnel on `8787`.
- **`umbrel.local` works on one device but not another.** Use the Tailscale `100.x` address or
  MagicDNS name instead of mDNS.
- **The TV cannot connect.** Confirm the TV and Umbrel are on the same VLAN and that the Umbrel
  firewall allows TCP `8787` from the LAN.
- **Geolocation is blocked in the web client.** Use Tailscale Serve/HTTPS, or open it on
  `localhost`.
- **Rooms vanished.** The relay is intentionally in-memory; a restart or six-hour TTL clears them.
