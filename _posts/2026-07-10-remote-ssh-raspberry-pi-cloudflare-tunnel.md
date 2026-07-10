---
title: "Remote SSH for Raspberry Pi with Cloudflare Tunnel"
layout: post
date: 2026-07-10 12:00
headerImage: false
tag:
- raspberry-pi
- cloudflare
- ssh
- security
star: false
category: blog
author: andreasrichardson
description: "How to expose your Raspberry Pi's SSH over the internet securely using Cloudflare Tunnel and Cloudflare Access, with no open ports."
---

Getting SSH access to a Raspberry Pi from outside my home network used to mean either port-forwarding on my router (inviting brute-force attempts on port 22) or running a VPN with its own infrastructure. Cloudflare Tunnel is cleaner: the Pi dials out to Cloudflare's network rather than listening for inbound connections, so there are no open ports at all.

Pairing the tunnel with [Cloudflare Access](https://developers.cloudflare.com/cloudflare-one/access-controls/) gates the SSH endpoint behind an identity layer: only authenticated users with the right policy can reach it. This guide covers the tunnel setup on the Pi, the Access application with identity provider and MFA, and the SSH client config to connect through it.

*Validated end to end in July 2026 with `cloudflared` 2026.7.1 on a Pi running Debian 13 (trixie).*

## Prerequisites

- Raspberry Pi with SSH enabled and a working internet connection
- [Cloudflare account](https://dash.cloudflare.com/sign-up) with [domain managed by Cloudflare DNS](https://developers.cloudflare.com/fundamentals/manage-domains/add-site/)
- [Cloudflare Zero Trust organisation](https://developers.cloudflare.com/cloudflare-one/setup/#2-create-a-zero-trust-organization) (the free tier covers this use case)

## Step 1: Install [`cloudflared`](https://github.com/cloudflare/cloudflared) on the Raspberry Pi

[Install the `cloudflared` daemon](https://github.com/cloudflare/cloudflared#installing-cloudflared), via the [Cloudflare Package Repository](https://pkg.cloudflare.com/index.html).

```bash
# add cloudflare gpg key
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null

# add the stable build for Debian to your apt repositories
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main' | sudo tee /etc/apt/sources.list.d/cloudflared.list

# install cloudflared
sudo apt-get update && sudo apt-get install cloudflared

# check installed version
cloudflared --version
```

Then authenticate:

```bash
cloudflared tunnel login
```

On a desktop this opens a browser window; on a headless Pi it prints a URL to open on any other device. Log in, select the zone to add the tunnel to, and click **Authorize**; `cloudflared` then writes a `cert.pem` credential to `~/.cloudflared/`.

## Step 2: Create the tunnel

```bash
cloudflared tunnel create raspberrypi
cloudflared tunnel info raspberrypi
```

[`tunnel create`](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/get-started/create-remote-tunnel/) registers a named tunnel with Cloudflare and writes a credentials JSON to `~/.cloudflared/<Tunnel-UUID>.json`. Keep that UUID to hand; you'll need it in the config file.[^1]

## Step 3: Set up [Cloudflare Access](https://www.cloudflare.com/sase/products/access/)

Cloudflare recommends creating the Access application *before* routing DNS to the tunnel, so the endpoint is protected from the moment it becomes reachable. (The Zero Trust dashboard is now branded **Cloudflare One**; the section names below have been stable, but expect the navigation to drift.)

### Add [identity providers](https://developers.cloudflare.com/cloudflare-one/integrations/identity-providers/)

From the [Cloudflare dashboard](https://dash.cloudflare.com) go to **Zero Trust (Cloudflare One) → Integrations → Identity providers → Add new identity provider** to configure how users authenticate. A few options I've found useful:

- **[One-time PIN](https://developers.cloudflare.com/cloudflare-one/integrations/identity-providers/one-time-pin/)**: the default option, where users receive a code by email.
- **[Cloudflare](https://developers.cloudflare.com/cloudflare-one/integrations/identity-providers/cloudflare/)**: authenticates users via their Cloudflare account. Optionally toggle *Restrict to account members* to limit access to people in your Cloudflare organisation.
- **[GitHub](https://developers.cloudflare.com/cloudflare-one/integrations/identity-providers/github/)**: requires a GitHub OAuth app first. Go to **GitHub Settings → Developer Settings → [OAuth Apps](https://github.com/settings/developers) → New OAuth app**, set the homepage URL to `https://<your-team-name>.cloudflareaccess.com` and the authorisation callback URL to `https://<your-team-name>.cloudflareaccess.com/cdn-cgi/access/callback`, then copy the Client ID and Client Secret into the Zero Trust GitHub IdP form.

### Add [independent MFA](https://developers.cloudflare.com/cloudflare-one/access-controls/policies/mfa-requirements/#independent-mfa) (optional)

For an extra layer on top of the IdP, go to **Zero Trust → Access controls → Access settings → Allow multi-factor authentication (MFA)** and enable MFA methods (e.g., **Biometrics** to use Passkeys), then tick **Apply global MFA settings by default** to enable MFA on all Applications and Policies.

### Create the [Access application](https://developers.cloudflare.com/cloudflare-one/access-controls/applications/http-apps/self-hosted-public-app/)

Go to **Zero Trust → Access controls → Applications → Create new application → Self-hosted and private**.

- Add a **Destination** for the SSH hostname (e.g. `raspberrypi-ssh.yourdomain.com`) and a second for the test hostname used in the tunnel config below (e.g. `raspberrypi-test.yourdomain.com`). Anything the tunnel exposes without an Access destination is reachable by the whole internet; an application supports up to five.  
**Pro Tip:** You can only have [1 level of subdomain for free](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/troubleshoot-tunnels/common-errors/#i-see-this-site-cant-provide-a-secure-connection).
- Add an [**Access policy**](https://developers.cloudflare.com/cloudflare-one/access-controls/policies/) with the inline builder, e.g. scoped to the emails that should have access (the selector is called *Emails*, plural):

  | Action | Rule type | Selector | Value           |
  | ------ | --------- | -------- | --------------- |
  | Allow  | Include   | Emails   | `you@email.com` |

  Name and save the policy. The default session duration is 24 hours, so expect to re-authenticate daily unless you change it.
- Copy the **Application Audience (AUD) tag** from the **Additional settings** tab; you'll need it for the tunnel config. (The application name is auto-generated from the first destination; rename it if you like.)

## Step 4: Write the tunnel config

Create [`~/.cloudflared/config.yml`](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/local-management/configuration-file/):

```yaml
tunnel: <Tunnel-UUID>
credentials-file: /home/pi/.cloudflared/<Tunnel-UUID>.json

ingress:
  # SSH traffic routed to port 22 (default)
  - hostname: raspberrypi-ssh.yourdomain.com
    service: ssh://localhost:22
    originRequest:
      # Validate Cloudflare Access JWTs before forwarding to the origin.
      # This rejects traffic that bypasses the Access application.
      access:
        required: true
        teamName: <your-team-name>
        audTag:
          - <Access-application-audience-tag>
  # test HTTP/S server to validate tunnel connection
  - hostname: raspberrypi-test.yourdomain.com
    service: hello_world
    originRequest:
      access:
        required: true
        teamName: <your-team-name>
        audTag:
          - <Access-application-audience-tag>
  # catch-all 404
  - service: http_status:404
```

- The [`originRequest.access`](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/configure-tunnels/origin-parameters/#access) block tells `cloudflared` to validate the Cloudflare Access JWT on every inbound request before forwarding it to the origin, so even traffic that reaches the tunnel while bypassing the Access layer is rejected.
- **The `access` block must sit under each individual ingress rule.** Despite what [the docs](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/local-management/configuration-file/#origin-configuration) suggest about top-level `originRequest` settings, a top-level `access` block is silently ignored and unauthenticated traffic is forwarded anyway (verified with 2026.7.1). Full explanation in [cloudflared#784](https://github.com/cloudflare/cloudflared/issues/784) and [cloudflare-docs#32006](https://github.com/cloudflare/cloudflare-docs/issues/32006).
- [Ingress rules](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/routing-to-tunnel/protocols/) are matched top-to-bottom; the catch-all `http_status:404` at the end is required.

Validate the config and dry-run a rule:

```bash
cloudflared tunnel ingress validate
cloudflared tunnel ingress rule ssh://raspberrypi-ssh.yourdomain.com
```

## Step 5: Route DNS

```bash
cloudflared tunnel route dns raspberrypi raspberrypi-ssh.yourdomain.com
cloudflared tunnel route dns raspberrypi raspberrypi-test.yourdomain.com
```

This automatically creates [CNAME records](https://developers.cloudflare.com/tunnel/routing/#dns-records) in your Cloudflare DNS zone pointing at the tunnel.

## Step 6: Test the tunnel

Before installing anything as a service, run the tunnel in the foreground:

```bash
cloudflared tunnel run raspberrypi
```

Within a few seconds the log should show several `Registered tunnel connection` lines. Then visit `https://raspberrypi-test.yourdomain.com` in a browser: after the Access login you should see the built-in hello world page ("Congrats! You created a tunnel!") served from the Pi.

If the test behaves oddly in the first minute: a bare `403` means the Access application hasn't propagated to the edge yet and the per-rule `access` block is rejecting the request itself (defence-in-depth doing its job); a `404` means the hostname fell through to the catch-all rule, so check the hostnames in `config.yml` match the DNS routes.

Stop the foreground tunnel with `Ctrl+C` before the next step.

## Step 7: Install as a system service

```bash
sudo cloudflared --config ~/.cloudflared/config.yml service install
```

This copies the config to `/etc/cloudflared/config.yml` and enables *and* starts a systemd service, so no separate `systemctl enable`/`start` is needed. Check it with:

```bash
systemctl status cloudflared
```

The explicit `--config` matters: under `sudo`, `cloudflared` searches root's config locations and won't find files in your home directory. (The shell expands `~` before `sudo` runs, so the command above passes the right path.)

The credentials JSON stays in `/home/pi/.cloudflared/`; future config edits go in `/etc/cloudflared/config.yml`, followed by:

```bash
sudo systemctl restart cloudflared
```

## Step 8: Connect from your client

[Install `cloudflared`](https://github.com/cloudflare/cloudflared#installing-cloudflared) on the machine you're connecting from (Mac: `brew install cloudflared`), then [configure `cloudflared` as proxy](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/use-cases/ssh/ssh-cloudflared-authentication/) in `~/.ssh/config`:

```
Host raspberrypi-ssh.yourdomain.com
  ProxyCommand cloudflared access ssh --hostname %h
```

(`cloudflared access ssh-config --hostname raspberrypi-ssh.yourdomain.com` prints exactly this block, with the absolute path to the binary filled in.)

Then connect as normal:

```bash
ssh pi@raspberrypi-ssh.yourdomain.com
```

The first connection opens a browser window for Cloudflare Access authentication (and SSH prompts you to accept the Pi's host key as usual); once authenticated, the session completes in the terminal. Subsequent connections within the Access session duration are seamless.

**Pro Tip:** if `cloudflared` isn't on your `$PATH` when SSH runs the `ProxyCommand`, use the absolute path: run `which cloudflared` and substitute the result.

---

The end result is a Pi that's reachable from anywhere, with no open ports on the router, no public IP dependency, and an identity-gated access layer with MFA that starts with the Pi on every boot. (Yes, I rebooted it to check.)

[^1]: `cloudflared tunnel info raspberrypi` shows the UUID alongside any active connectors. You can also find it in the filename of the credentials JSON under `~/.cloudflared/`.
