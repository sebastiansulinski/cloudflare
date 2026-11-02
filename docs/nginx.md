[Contents](../README.md)

# nginx on the server

Every request reaches nginx from cloudflared on `127.0.0.1`, over plain HTTP. Without the
changes below:

- **The site loops:** a port 80 block that redirects to HTTPS sends the browser back through
  the tunnel, which arrives on port 80 again.
- **Every visitor looks like `127.0.0.1`:** rate limits (contact forms, logins) are shared
  by everyone, and logs record nothing useful.
- **PHP thinks the request is insecure:** Laravel builds `http://` links for assets and
  routes (the page renders unstyled), and an application that only sends
  `Strict-Transport-Security` on secure requests never sends it.

## The changes for each site

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

   Remove `add_header X-Content-Type-Options "nosniff";` too when the application sends it
   itself, as the Laravel sites here do: otherwise the header is sent twice. Keep it for a site
   whose application does not. Keep any `Strict-Transport-Security` line unless the application
   sends the header itself. A `location` block with its own
   `add_header` drops all of the server block's headers inside that location.

3. **Trust the visitor address only from cloudflared**, in the server block (below its
   `add_header` lines, or where they were):

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

## Every site listens the same way

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

## The default server

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

## Certificates and closing the public ports

After the 48-hour overlap (see [Switch the nameservers](move-dns.md#switch-the-nameservers)), check the site's access log
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

Previous: [Serve a site through the tunnel](serve-site-through-tunnel.md) | Next: [Forge nginx template](forge-nginx-template.md)
