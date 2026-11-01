# Cloudflare runbook

How domains and sites are moved to and run on Cloudflare: moving a domain's Domain Name
System (DNS) and its registration, the settings every zone gets, serving a site from a server
through a Cloudflare Tunnel, configuring nginx (by hand and through a Forge nginx template),
and domains that only redirect to another domain.

Every command block is self-contained: set the variables on its first lines, then copy the
whole block. Dashboard paths are as the Cloudflare and Forge dashboards showed them on
2026-11-01. They move things around; Cloudflare's search box at the top of the dashboard
finds a setting by name.

```
visitor ──https──> Cloudflare edge ──tunnel (outbound from the server)──> cloudflared
                                                                            │ http, Host header kept
                                                                            v
                                                              nginx on port 80 ──> PHP
```

Cloudflare ends the encrypted connection, Transport Layer Security (TLS), at its edge and
presents its own certificate to visitors. Cloudflare's dashboard still labels it "SSL", after
the older Secure Sockets Layer. The server makes an outbound connection to Cloudflare, so nothing has to listen on
ports 80 or 443 to the internet, and the server needs no certificate.

## Contents

1. [Before you start](#1-before-you-start)
2. [Move a domain's DNS to Cloudflare](#2-move-a-domains-dns-to-cloudflare)
   - [2.1 Record the current records](#21-record-the-current-records)
   - [2.2 Switch off DNSSEC at the old registrar](#22-switch-off-dnssec-at-the-old-registrar)
   - [2.3 Add the zone](#23-add-the-zone)
   - [2.4 Check the imported records](#24-check-the-imported-records)
   - [2.5 Switch the nameservers](#25-switch-the-nameservers)
3. [Transfer the registration](#3-transfer-the-registration)
   - [3.1 Domains ending in .uk](#31-domains-ending-in-uk)
   - [3.2 .com and other generic domains](#32-com-and-other-generic-domains)
   - [3.3 After the transfer: DNSSEC on](#33-after-the-transfer-dnssec-on)
4. [Zone settings for every domain](#4-zone-settings-for-every-domain)
5. [The tunnel (once per server)](#5-the-tunnel-once-per-server)
6. [Serve a site through the tunnel](#6-serve-a-site-through-the-tunnel)
   - [6.1 Routes](#61-routes)
   - [6.2 DNS records for the site](#62-dns-records-for-the-site)
   - [6.3 Redirect www to the bare domain](#63-redirect-www-to-the-bare-domain)
7. [nginx on the server](#7-nginx-on-the-server)
   - [7.1 The changes for each site](#71-the-changes-for-each-site)
   - [7.2 Every site listens the same way](#72-every-site-listens-the-same-way)
   - [7.3 The default server](#73-the-default-server)
   - [7.4 Certificates and closing the public ports](#74-certificates-and-closing-the-public-ports)
8. [Forge nginx template](#8-forge-nginx-template)
9. [Domains that only redirect](#9-domains-that-only-redirect)
   - [9.1 DNS records](#91-dns-records)
   - [9.2 The redirect rule](#92-the-redirect-rule)
10. [Checks](#10-checks)
11. [Troubleshooting](#11-troubleshooting)
12. [Sources](#12-sources)

---

## 1. Before you start

Collect for each domain:

- [ ] Where it is registered (Namecheap, for the domains so far) and a login there.
- [ ] Whether it serves a site, only redirects, or does nothing.
- [ ] Whether it handles email, and with whom (Google Workspace, Amazon Simple Email Service).
- [ ] For a site: the server it runs on, and whether that server already has a tunnel.

Order of work, so nothing goes offline:

1. Move the DNS ([section 2](#2-move-a-domains-dns-to-cloudflare)), with the site or redirect
   ready in Cloudflare before the nameservers change.
2. Only then transfer the registration ([section 3](#3-transfer-the-registration)). Cloudflare
   needs an active zone first, and the transfer itself changes nothing visitors see.

---

## 2. Move a domain's DNS to Cloudflare

### 2.1 Record the current records

Cloudflare's import only finds records it can guess the names of, so keep a copy of what the
current provider serves. Record types used below: `A` and `AAAA` (addresses), `CNAME`
(canonical name: an alias for another name), `MX` (mail exchange), `TXT` (text) and `CAA`
(which certificate authorities may issue for the domain). The old provider's dashboard (Namecheap: **Domain List, Manage,
Advanced DNS**) is the full list; this shows the common ones:

```bash
DOMAIN=example.co.uk
for t in NS A AAAA MX TXT CAA; do echo "== $t"; dig +short $t "$DOMAIN"; done
for h in www _dmarc google._domainkey; do echo "== $h"; dig +short "$h.$DOMAIN"; dig +short TXT "$h.$DOMAIN"; done
```

Look out for records with random-looking names, such as the three Amazon Simple Email Service
signing records ending in `._domainkey`: no lookup can list them.

### 2.2 Switch off DNSSEC at the old registrar

Domain Name System Security Extensions (DNSSEC) signs the domain's records, and the registry
publishes a Delegation Signer (DS) record that points at the old provider's keys. If the nameservers
change while it is on, every resolver that validates signatures fails to look the domain up:
the site and email both stop.

1. Namecheap: **Domain List, Manage, Advanced DNS, DNSSEC**: off.
2. Wait until the registry and the big resolvers no longer have the record. The registry
   caches it for 1 hour on `.uk` and 24 hours on `.com`, so wait at least that long after
   switching it off.

```bash
DOMAIN=example.co.uk
REGISTRY=$(dig +short NS "${DOMAIN#*.}" | head -1)
echo "== registry $REGISTRY"; dig +nocmd DS "$DOMAIN" @"$REGISTRY" +norec +noall +answer
for r in 1.1.1.1 8.8.8.8 9.9.9.9; do echo "== $r"; dig +short DS "$DOMAIN" @$r; done
```

Nothing printed under any heading means the record is gone. `whois` showing
`DNSSEC: unsigned` says the same.

### 2.3 Add the zone

1. Cloudflare, account home: **Add a domain** (the button may read **Onboard a domain** or
   **Connect a domain**).
2. Enter the domain, let it scan for existing records, choose the **Free** plan.
3. Note the two nameservers Cloudflare shows (`<name>.ns.cloudflare.com`). Don't switch yet.

The zone shows **Pending** until the nameservers change. Records, routes, rules and
settings can all be set up while it is pending.

### 2.4 Check the imported records

**Domain, DNS, Records.** Compare with the list from [2.1](#21-record-the-current-records):

- **Email records:** `MX`, the Sender Policy Framework `TXT` on the bare domain, DomainKeys
  Identified Mail (`google._domainkey` or others), `_dmarc`, and verification `TXT` records
  (Google Search Console and the like). All **DNS only** (grey cloud); `MX` cannot be proxied.
- **One key, one record.** A long DomainKeys Identified Mail key can arrive as two separate
  `TXT` records (one starting `v=DKIM1; k=rsa; p=...` and one holding the rest). Receiving
  servers cannot read a key split across records. Delete both and add one `TXT` record with
  the whole value; Cloudflare splits long values into chunks inside the record itself. For
  Google Workspace, copy the value from
  [Authenticate email](https://admin.google.com/ac/apps/gmail/authenticateemail) (super
  administrator only): don't click **Generate new record**, which replaces the key.
- **Web records:** for a site, delete the imported `A` and `CNAME` records for the bare
  domain and `www` (the tunnel routes replace them, [section 6](#6-serve-a-site-through-the-tunnel)).
  For a domain that only redirects, delete everything and follow
  [section 9](#9-domains-that-only-redirect).
- **Wildcards:** delete an imported `*` record unless something really uses it.

Apply the [zone settings](#4-zone-settings-for-every-domain) now too.

### 2.5 Switch the nameservers

1. Namecheap: **Domain List, Manage, Nameservers, Custom DNS**: the two Cloudflare
   nameservers. Save.
2. Cloudflare emails when the zone becomes **Active**, usually within minutes.

```bash
DOMAIN=example.co.uk
REGISTRY=$(dig +short NS "${DOMAIN#*.}" | head -1)
dig +nocmd NS "$DOMAIN" @"$REGISTRY" +norec +noall +authority
```

It lists the two Cloudflare nameservers once the registry has the change.

**The overlap.** The registry tells resolvers to keep the old nameservers for up to 48 hours,
and the old provider keeps answering with the old records. For those 48 hours some visitors
still go straight to the server's address, so leave the server's public ports and
certificate as they are until the overlap ends ([7.4](#74-certificates-and-closing-the-public-ports)).

---

## 3. Transfer the registration

Only once the zone is **Active**. Visitors notice nothing: the DNS is already Cloudflare's.
A domain transferred to Cloudflare cannot move to another registrar for 60 days.

### 3.1 Domains ending in .uk

`.uk` domains move by changing the domain's Internet Provider Security (IPS) tag at Nominet,
the `.uk` registry, not by an authorisation code. The order matters:

1. Cloudflare, account home: **Domain Registration, Transfer Domains**, select the domain and
   complete checkout. No fee, and no year is added.
2. **Only after checkout**, set the domain's IPS tag to the value below at the current
   registrar (Namecheap: under the domain's **Sharing & Transfer**, or ask its live chat
   support). A tag change before checkout makes Cloudflare reject the transfer.

   ```
   CLOUDFLARE
   ```

3. Cloudflare completes it when Nominet records the change, usually within hours. Ask the
   registrar again if nothing happens within 24 hours; the transfer cancels itself after
   30 days.

### 3.2 .com and other generic domains

1. Namecheap: turn the domain's **Registrar Lock** off, then request the authorisation code
   under **Sharing & Transfer**. It arrives by email.
2. Cloudflare: **Domain Registration, Transfer Domains**, select the domain, enter the code,
   pay. The price is one year at Cloudflare's cost, added to the current expiry date.
3. Approve the transfer from Namecheap's confirmation email. Without approval it completes on
   its own after about 5 days.

```bash
DOMAIN=example.com
whois "$DOMAIN" | grep -iE "registrar:|status|dnssec" | sort -u
```

`Registrar: Cloudflare, Inc.` means it is done.

### 3.3 After the transfer: DNSSEC on

**Domain, DNS, Settings, DNSSEC: enable.** With Cloudflare as registrar, it publishes the
signing record at the registry itself, with nothing to copy.

---

## 4. Zone settings for every domain

In the zone (account home, then the domain):

| Setting | Where | Value | Cloudflare's default |
|---|---|---|---|
| SSL/TLS encryption mode | **SSL/TLS, Overview** | Full (strict) | varies |
| Always Use HTTPS | **SSL/TLS, Edge Certificates** | On | Off |
| Minimum TLS Version | same page | TLS 1.2 | TLS 1.0 |
| HTTP Strict Transport Security (HSTS) | same page | Not enabled | Not enabled |
| Email Address Obfuscation | **Security, Settings**, group **Client side abuse** | Off | On |
| Bot fight mode | **Security, Settings**, group **Bot traffic** | Off | Off |
| Rocket Loader | **Speed, Settings**, tab **Content Optimization** | Off | Off |

Why:

- **Encryption mode** does not affect tunnel traffic (the connection from Cloudflare to
  cloudflared is encrypted on its own, and the route's service address decides the rest).
  Full (strict) is the safe value should a record ever point straight at a server.
- **Always Use HTTPS** sends every `http://` request to `https://` at Cloudflare's edge, so
  nginx never needs its own redirect (and must not have one: [7.1](#71-the-changes-for-each-site)).
- **Minimum TLS Version 1.2.** TLS 1.0 and 1.1 are retired; every
  browser in use supports 1.2.
- **Strict Transport Security stays off in Cloudflare.** The site sends its own header (from
  nginx or the application). Cloudflare's setting covers the whole zone, and with "include
  subdomains" or "preload" it forces HTTPS on every subdomain for the whole period, which
  preload makes very hard to undo.
- **Email Address Obfuscation, Bot fight mode and Rocket Loader inject Cloudflare scripts**
  into the page. A site with a strict Content Security Policy blocks them; obfuscation then
  shows "[email protected]" wherever the page has an email address.
- **Caching** stays at the default: static files only, never HTML, which carries form tokens.

---

## 5. The tunnel (once per server)

One tunnel per server, whatever the number of sites and domains on it. A tunnel belongs to
the account, not to a zone, so it carries hostnames from any of the account's domains.

1. Cloudflare: **Tunnels** (search for it if the menu has moved), **Create Tunnel**, type
   **Cloudflared**.
2. **Name:** the server's name. Only a label.
3. On the server, as `forge`, run the install commands the dashboard shows (Debian, 64-bit:
   Forge runs Ubuntu). The last one is `sudo cloudflared service install <token>`. The token
   lets anyone run the tunnel: paste it straight from the dashboard into the terminal, never
   into a file, chat or repository.
4. Check:

   ```bash
   sudo systemctl status cloudflared
   ```

   It shows `active (running)`, and the dashboard shows the connector **Healthy**.

---

## 6. Serve a site through the tunnel

### 6.1 Routes

**Tunnels**, the server's tunnel, tab **Routes**, **Add route**, **Published application**.
One route for the bare domain, one for `www`:

| Field | Bare domain | www |
|---|---|---|
| Subdomain | empty | `www` |
| Domain | `example.co.uk` | `example.co.uk` |
| Path | empty | empty |
| Service URL | `http://localhost:80` | `http://localhost:80` |

Leave **Additional application settings** on their defaults: cloudflared passes the
visitor's `Host` header through, so nginx picks the site by `server_name`.

Plain HTTP on port 80 is the standard here: the connection from cloudflared to nginx never
leaves the server, so encrypting it adds nothing, and there is no certificate to renew. It
needs the nginx changes in [7.1](#71-the-changes-for-each-site).

<details>
<summary>The alternative: <code>https://localhost:443</code></summary>

Only for a site whose nginx must keep its HTTPS block (a certificate already in place, no
nginx changes wanted). Each route then needs **Additional application settings, TLS, Origin
Server Name** set to that route's full hostname (`example.co.uk`, `www.example.co.uk`), and
**No TLS Verify** left off. Without it cloudflared asks nginx for `localhost`, gets the default
server's certificate, rejects it, and visitors see a 502 error. The certificate then has to
keep renewing, and Let's Encrypt's HTTP validation would have to pass through the tunnel.

</details>

### 6.2 DNS records for the site

Adding a route creates a proxied record pointing at the tunnel (type **Tunnel**, or `CNAME`
to `<tunnel id>.cfargotunnel.com`). Check **DNS, Records**:

- The bare domain and `www` each have one, orange cloud.
- If a route's record is missing (the **Recommendations** box says "Visitors cannot reach
  www..."), add it: **Add record**, type **Tunnel** and pick the tunnel, or type `CNAME`,
  name `www`, target `<tunnel id>.cfargotunnel.com` (the id is on the tunnel's
  **Overview**), **Proxied**.
- A route cannot be saved while an `A` or `CNAME` record with the same name exists: delete
  the old record first. That deletion is the switch-over.
- The information icon on the bare domain's record ("CNAME flattening") is expected and needs
  nothing.

### 6.3 Redirect www to the bare domain

Either in nginx (a `www` server block with `return 301 https://example.co.uk$request_uri;`)
or, tidier, in Cloudflare so the request never reaches the server: **Rules, Redirect Rules,
Create rule**, the template **Redirect from WWW to root**, or by hand:

- **When:** custom filter, **Hostname**, **equals**, `www.example.co.uk`
- **Then:** Dynamic, expression below, **301**, **Preserve query string** on

```
concat("https://example.co.uk", http.request.uri.path)
```

---

## 7. nginx on the server

Every request reaches nginx from cloudflared on `127.0.0.1`, over plain HTTP. Without the
changes below:

- **The site loops:** a port 80 block that redirects to HTTPS sends the browser back through
  the tunnel, which arrives on port 80 again.
- **Every visitor looks like `127.0.0.1`:** rate limits (contact forms, logins) are shared
  by everyone, and logs record nothing useful.
- **PHP thinks the request is insecure:** Laravel builds `http://` links for assets and
  routes (the page renders unstyled), and an application that only sends
  `Strict-Transport-Security` on secure requests never sends it.

### 7.1 The changes for each site

In the site's nginx configuration (Forge: the site's **Edit Nginx configuration**; on a site
split into two files, the changes go in the one with the headers, `location` blocks and the
PHP block):

1. **No redirect on port 80.** The server block on port 80 serves the site itself. Delete any
   `return 301 https://...` for the site's own hostname. On Forge, a site with an active
   certificate gets a redirecting port 80 block, so remove the certificate (or create the
   site without one).

2. **Remove two obsolete or conflicting headers** (Laravel sites here send
   `X-Frame-Options: DENY` themselves, and two different values let browsers ignore both;
   `X-XSS-Protection` is obsolete):

   ```nginx
   add_header X-Frame-Options "SAMEORIGIN";
   add_header X-XSS-Protection "1; mode=block";
   ```

   Keep `add_header X-Content-Type-Options "nosniff";`. Keep any `Strict-Transport-Security`
   line unless the application sends the header itself. A `location` block with its own
   `add_header` drops all of the server block's headers inside that location.

3. **Trust the visitor address only from cloudflared**, directly below the `nosniff` line:

   ```nginx
   # Only cloudflared reaches nginx (ports 80 and 443 are closed), so its headers are trusted.
   set_real_ip_from 127.0.0.1;
   set_real_ip_from ::1;
   real_ip_header CF-Connecting-IP;
   ```

4. **Tell PHP the request is HTTPS**, inside the `location ~ \.php$` block, after the
   `include` line(s) for the FastCGI parameters:

   ```nginx
   fastcgi_param HTTPS on;
   ```

5. **Laravel applications that generate `robots.txt`:** the exact-match block answers 404
   before Laravel, so let it fall through:

   ```nginx
   location = /robots.txt  { try_files $uri /index.php?$query_string; access_log off; log_not_found off; }
   ```

Test and reload (Forge reloads on save):

```bash
sudo nginx -t && sudo systemctl reload nginx
```

Check from the server, with the site's hostname (expects `200`):

```bash
HOST=example.co.uk
curl -s -o /dev/null -w '%{http_code}\n' -H "Host: $HOST" http://127.0.0.1/
```

Laravel applications here trust no proxy headers themselves (no `trustProxies`): nginx's
`real_ip` lines replace that, and are safe only because they trust nothing but `127.0.0.1`.

### 7.2 Every site listens the same way

All sites share port 80; nginx tells them apart by `server_name`, from the `Host` header
cloudflared passes on. Keep every server block on the same form:

```nginx
listen 80;
listen [::]:80;
```

Never give one site `listen 127.0.0.1:80;`. nginx matches the address before the name: once
one block listens on `127.0.0.1:80` explicitly, requests arriving on `127.0.0.1:80` are only
matched against blocks with that exact `listen`, so every other site's traffic lands on that
one site.

### 7.3 The default server

A request with an unknown hostname should get nothing, not whichever site nginx loaded first.
Check one block on port 80 is the default:

```bash
sudo nginx -T 2>/dev/null | grep -n "default_server"
```

If none is, add a catch-all (Forge has its own `default` site for this):

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 444;
}
```

### 7.4 Certificates and closing the public ports

After the 48-hour overlap of [2.5](#25-switch-the-nameservers), check the site's access log
for requests that still arrive directly (they show the visitor's own address and no
`CF-Connecting-IP`). Once only bots arrive:

1. **Delete the site's HTTPS (443) server block and its certificate.** For a Let's Encrypt
   certificate, also its renewal, which would otherwise start failing:

   ```bash
   DOMAIN=example.co.uk
   sudo certbot delete --cert-name "$DOMAIN"
   ```

   On Forge, remove the certificate under the site's **SSL** instead.

2. **Close ports 80 and 443 to the internet** (Forge: the server's **Network** firewall
   rules: delete the rules for 80 and 443). cloudflared reaches nginx from the server itself,
   so the site keeps working. Port 22: allow only your own address, Forge's and Envoyer's.

3. **Check from a network the firewall doesn't allow** (not your allowed address or virtual
   private network):

   ```bash
   SERVER=203.0.113.10
   HOST=example.co.uk
   curl -m 5 -s -o /dev/null -w '%{http_code}\n' -H "Host: $HOST" "http://$SERVER/"
   curl -m 5 -kI "https://$SERVER"
   ```

   Both must time out. An answer means the port is open, and the site is reachable around
   Cloudflare (and around Cloudflare Access, for a site behind it). It also means
   `fastcgi_param HTTPS on` marks a plain request as secure.

---

## 8. Forge nginx template

A Forge nginx template carries the [7.1](#71-the-changes-for-each-site) changes, so a site
created with it needs no hand edits.

**Create it once per server:** the server's dashboard, **Nginx Templates**, **New template**.
Forge fills in its default template; name the copy `Cloudflare Tunnel` and edit it:

1. Delete the `X-Frame-Options` and `X-XSS-Protection` lines.
2. Add the three `real_ip` lines below `add_header X-Content-Type-Options "nosniff";`.
3. Add `fastcgi_param HTTPS on;` inside the `location ~ \.php$` block, after its `include`
   line(s).
4. For Laravel sites that generate `robots.txt`: replace the `robots.txt` line as in
   [7.1](#71-the-changes-for-each-site).
5. Leave the `listen`, `server_name`, `root` and socket lines on Forge's variables, and
   leave Forge's `# FORGE ... (DO NOT REMOVE!)` comment lines in place: Forge uses them to
   find where its own includes go.

The changes are the same on every site; only Forge's variables differ. They must be written
exactly as below, with no spaces inside the braces:

| Variable | Becomes |
|---|---|
| `{{DOMAINS}}` | the site's domains, aliases included: `example.co.uk www.example.co.uk` |
| `{{SITE}}` | the site's name, without aliases: `example.co.uk` |
| `{{SITE_ID}}` | the site's id in Forge: `12345` |
| `{{PATH}}` | the web directory's full path: `/home/<user>/example.co.uk/public` |
| `{{ROOT_PATH}}` | the site's root: `/home/<user>/example.co.uk` |
| `{{DIRECTORY}}` | the web directory: `/public` |
| `{{PORT}}` | `80` (with `default_server` for the `default` site) |
| `{{PORT_V6}}` | `[::]:80` (likewise) |
| `{{PROXY_PASS}}` | the PHP socket: `unix:/var/run/php/php8.4-fpm.sock` |
| `{{USER}}` | the site's user: `forge`, or the isolated user |
| `{{SERVER_PUBLIC_IP}}`, `{{SERVER_PRIVATE_IP}}` | the server's addresses |

**Use it for each new site:** choose the template when creating the site in Forge, with **no
certificate** (Let's Encrypt cannot validate through the tunnel, and Cloudflare provides
HTTPS) and **no DNS change** (ignore any Forge prompt to point the domain at the server: an
`A` record bypasses the tunnel). Then add the site's routes ([6.1](#61-routes)).

Things to know:

- **Editing a template does not change sites already created with it.** Edit those by hand
  ([7.1](#71-the-changes-for-each-site)).
- **An invalid template stops nginx**, taking every site on the server down. After creating
  the first site from a new or edited template, check straight away:

  ```bash
  sudo nginx -t
  ```

- Forge may rewrite a site's configuration when some of its settings change. After any
  change to a site in Forge, check the [7.1](#71-the-changes-for-each-site) lines are still
  there:

  ```bash
  sudo nginx -T 2>/dev/null | grep -nE "real_ip_header|fastcgi_param HTTPS|X-Frame|X-XSS"
  ```

---

## 9. Domains that only redirect

A domain that sends every request to another domain (an old brand, a `.com` beside a
`.co.uk`) needs no server at all: Cloudflare answers with the redirect at its edge.

### 9.1 DNS records

Delete every imported record, then add exactly these:

| Type | Name | Content | Proxy | Why |
|---|---|---|---|---|
| `A` | `@` | `192.0.2.1` | **Proxied** | placeholder so Cloudflare answers for the bare domain |
| `A` | `www` | `192.0.2.1` | **Proxied** | placeholder for `www` |
| `MX` | `@` | `.` (priority `0`) | DNS only | "null MX": the domain accepts no email |
| `TXT` | `@` | `v=spf1 -all` | DNS only | no server may send email as this domain |
| `TXT` | `_dmarc` | `v=DMARC1; p=reject;` | DNS only | receivers reject anything claiming to be from it |

- `192.0.2.1` is an address reserved for documentation, which never reaches anything. A
  redirect rule only runs for proxied hostnames, and the placeholder is what makes them
  proxied. If the rule ever stops matching, visitors get a Cloudflare 522 or 523 error
  (Cloudflare trying to reach the placeholder), which shows in a browser as a timeout.
- The three email records are for a domain that **handles no email**. If it does, keep its
  real `MX`, Sender Policy Framework, DomainKeys Identified Mail and Domain-based Message
  Authentication, Reporting and Conformance (DMARC) records instead.

### 9.2 The redirect rule

**Rules, Redirect Rules, Create rule:**

- **Rule name:** `Redirect to example.co.uk`
- **If incoming requests match:** **Custom filter expression**: field **Hostname**,
  operator **is in**, and the two hostnames **as two separate values**: type one, press
  **Enter** so it becomes its own chip, then the other.
- **Then:** **Dynamic**, the expression below, status **301**, **Preserve query string**
  on.

The expression preview must show two separately quoted names with a space between them:

```
(http.host in {"old-domain.com" "www.old-domain.com"})
```

Or click **Edit expression** and paste that line, with the real names. One quoted string
containing a comma (`{"old-domain.com,www.old-domain.com"}`) matches nothing, so every
request falls through to the placeholder and times out.

The target, sending every path to the same path on the new domain:

```
concat("https://example.co.uk", http.request.uri.path)
```

Point it at the final address: if `example.co.uk` itself redirects to
`www.example.co.uk`, use that, so visitors make one hop instead of two.

Each zone has its own rules. A rule created in one domain's zone does nothing for another.

Check (expects `301` to the same path on the target, query string kept, for both
hostnames and both schemes):

```bash
FROM=old-domain.com
for u in "http://$FROM/" "https://$FROM/about?x=1" "http://www.$FROM/" "https://www.$FROM/about?x=1"; do
  printf "%-45s " "$u"; curl -s -o /dev/null -m 20 -w "%{http_code} %{redirect_url}\n" "$u"
done
```

---

## 10. Checks

After a site goes live through the tunnel. Each block sets its variables first.

**DNS and registration:**

```bash
DOMAIN=example.co.uk
REGISTRY=$(dig +short NS "${DOMAIN#*.}" | head -1)
echo "== nameservers at the registry"; dig +nocmd NS "$DOMAIN" @"$REGISTRY" +norec +noall +authority | awk '{print $5}'
echo "== signing record at the registry"; dig +nocmd DS "$DOMAIN" @"$REGISTRY" +norec +noall +answer | grep DS
for t in A MX TXT; do echo "== $t"; dig +short $t "$DOMAIN"; done
echo "== www"; dig +short A "www.$DOMAIN"
```

**Redirects and status** (expects `301` to `https://` for the first three, `200`, then
`404`):

```bash
DOMAIN=example.co.uk
for u in "http://$DOMAIN/" "http://www.$DOMAIN/" "https://www.$DOMAIN/" "https://$DOMAIN/" "https://$DOMAIN/does-not-exist"; do
  printf "%-45s " "$u"; curl -s -o /dev/null -m 20 -w "%{http_code} %{redirect_url}\n" "$u"
done
```

**Headers** (expects `server: cloudflare`, one `x-frame-options`, no `x-xss-protection`,
cookies marked `secure`):

```bash
DOMAIN=example.co.uk
curl -sI "https://$DOMAIN/" | grep -iE "^(server|x-frame|x-xss|strict-transport|x-content-type|content-security|set-cookie)" | cut -c1-160
```

**No insecure links in the page** (expects nothing):

```bash
DOMAIN=example.co.uk
curl -s "https://$DOMAIN/" | grep -oE '(src|href|content)="http://[^"]+'
```

**Real visitor addresses** (on the server): the site's access log shows visitors'
addresses, not `127.0.0.1`.

**Email signing key is one record** (expects one line, starting `"v=DKIM1;`):

```bash
DOMAIN=example.co.uk
dig +short TXT "google._domainkey.$DOMAIN"
```

---

## 11. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Lookups fail for everyone right after the nameserver change | DNSSEC was still on at the old registrar | [2.2](#22-switch-off-dnssec-at-the-old-registrar); switching it off now fixes it once the cached signing record expires |
| `https://example.co.uk/` redirects to itself (301 to the same address, page signed "nginx") | nginx's port 80 block redirects to HTTPS; the tunnel arrives on port 80 | [7.1](#71-the-changes-for-each-site) step 1 |
| 502 error with an `https://localhost:443` route | No **Origin Server Name** on the route, so the certificate doesn't match | Set it ([6.1](#61-routes)), or switch the route to `http://localhost:80` |
| Redirect-only domain times out, or shows 522 or 523 | The redirect rule isn't matching: both hostnames in one value, rule in another zone, rule not deployed, or the wrong rule type | [9.2](#92-the-redirect-rule) |
| "Visitors cannot reach www" in **Recommendations** | The `www` route created no record | Add it ([6.2](#62-dns-records-for-the-site)) |
| Adding a route fails: record already exists | An imported `A` or `CNAME` has the same name | Delete it, save the route again |
| Works for others, times out on your own computer | Your computer still caches the old address | Flush macOS's cache (below); in Chrome also `chrome://net-internals/#dns`, **Clear host cache** |
| Page unstyled, `http://` links in the page | PHP doesn't know the request is HTTPS | [7.1](#71-the-changes-for-each-site) step 4 |
| A contact form's limit hits every visitor at once; logs show `127.0.0.1` | nginx doesn't take the address from cloudflared | [7.1](#71-the-changes-for-each-site) step 3 |
| Two `x-frame-options` headers | nginx adds `SAMEORIGIN`, the application `DENY` | [7.1](#71-the-changes-for-each-site) step 2 |
| "[email protected]" instead of an email address | Email Address Obfuscation is on | [Section 4](#4-zone-settings-for-every-domain) |
| Google mail fails DomainKeys Identified Mail checks | The key is split across two `TXT` records | [2.4](#24-check-the-imported-records) |
| Certificate renewal failure emails after the move | The old certificate is still set to renew | [7.4](#74-certificates-and-closing-the-public-ports) |
| One site's traffic lands on another site | One server block listens on `127.0.0.1:80`, the rest on `80` | [7.2](#72-every-site-listens-the-same-way) |
| Zone active but the site and email are gone | The zone has no records (nothing was imported or added) | Re-create them from the old provider's list ([2.1](#21-record-the-current-records)) |

Flush macOS's DNS cache:

```bash
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
```

Test a hostname through Cloudflare regardless of what your computer caches:

```bash
DOMAIN=example.co.uk
IP=$(dig +short A "$DOMAIN" @1.1.1.1 | head -1)
curl -s -o /dev/null -w "%{http_code} %{redirect_url}\n" --resolve "$DOMAIN:443:$IP" "https://$DOMAIN/"
```

---

## 12. Sources

- [Cloudflare: transfer a .uk domain](https://developers.cloudflare.com/registrar/get-started/transfer-uk-domain/)
- [Cloudflare: .uk domains](https://developers.cloudflare.com/registrar/top-level-domains/uk-domains/)
- [Cloudflare: transfer to Cloudflare](https://developers.cloudflare.com/registrar/domain-transfers/transfer-to-cloudflare)
- [Cloudflare: Tunnel with HTTPS origins](https://developers.cloudflare.com/tunnel/troubleshooting/https-origins/)
- [Cloudflare: create a single redirect in the dashboard](https://developers.cloudflare.com/rules/url-forwarding/single-redirects/create-dashboard/)
- [Cloudflare: redirect all requests to another domain](https://developers.cloudflare.com/rules/url-forwarding/examples/redirect-all-another-domain/)
- [Cloudflare: Always Use HTTPS](https://developers.cloudflare.com/ssl/edge-certificates/additional-options/always-use-https/)
- [Cloudflare: Minimum TLS Version](https://developers.cloudflare.com/ssl/edge-certificates/additional-options/minimum-tls/)
- [Cloudflare: TLS protocols](https://developers.cloudflare.com/ssl/reference/protocols/)
- [Laravel Forge: nginx templates](https://laravel.com/forge/docs/servers/nginx-templates)
- [nginx: how nginx processes a request](https://nginx.org/en/docs/http/request_processing.html)
- [nginx: the real IP module](https://nginx.org/en/docs/http/ngx_http_realip_module.html)
