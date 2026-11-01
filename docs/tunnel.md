[Contents](../README.md)

# The tunnel (once per server)

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

Previous: [Zone settings for every domain](zone-settings.md) | Next: [Serve a site through the tunnel](serve-site-through-tunnel.md)
