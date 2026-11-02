[Contents](../README.md)

# Serve an MTA-STS policy from a Worker

Mail Transfer Agent Strict Transport Security (MTA-STS) tells servers sending mail to a
domain that they must use an encrypted connection, Transport Layer Security (TLS), to the
servers it lists. It needs a small text file at a fixed address:

```
https://mta-sts.<mail domain>/.well-known/mta-sts.txt
```

That is the only thing the `mta-sts` hostname is for, so it needs no server: a Cloudflare
Worker returns the file at Cloudflare's edge. Below, DNS is the Domain Name System, `MX` a
mail exchange record and `TXT` a text record.

The example is `mail.mailtransporter.co.uk`, the custom sending address ("MAIL FROM"
domain) of an Amazon Simple Email Service (SES) identity, so its `MX` points at SES's bounce
endpoint. Replace the names and the `mx:` line with the domain's own.

## The records

| Type | Name | Content | Proxy |
|---|---|---|---|
| `TXT` | `_mta-sts.mail` | `v=STSv1; id=2026110202` | DNS only |
| `TXT` | `_smtp._tls.mail` | `v=TLSRPTv1; rua=mailto:report@reports.mailtransporter.co.uk` | DNS only |

- `_mta-sts` announces that a policy exists. Sending servers fetch the file again only when
  `id` changes, so **change the `id` every time the policy changes** (any letters and
  digits; the date plus a counter keeps it readable).
- `_smtp._tls` is the Simple Mail Transfer Protocol (SMTP) TLS reporting record: the address
  that receives daily reports of failed encrypted connections. Where they go is
  [Reports without an inbox](#reports-without-an-inbox).
- No record for the `mta-sts` hostname itself: the Worker creates it.

## The Worker

1. Cloudflare account home (not inside a domain): **Compute (Workers), Workers & Pages,
   Create**, **Start with Hello World**. Name it after the domain, for example
   `mta-sts-mailtransporter`, and **Deploy**.
2. **Edit code**, replace everything with the code below, **Deploy**.

```js
const POLICY = `version: STSv1
mode: enforce
mx: feedback-smtp.eu-west-1.amazonses.com
max_age: 604800
`;

export default {
  fetch(request) {
    const { pathname } = new URL(request.url);
    if (pathname !== '/.well-known/mta-sts.txt') {
      return new Response('Not found', { status: 404 });
    }
    return new Response(POLICY, {
      headers: { 'content-type': 'text/plain; charset=utf-8' },
    });
  },
};
```

- `mode`: `testing` only reports failures; `enforce` makes senders refuse to deliver without
  a valid certificate on a listed server. Start with `testing` only if someone will read
  the reports.
- `mx`: one line per mail server, exactly as the `MX` records name them (a `*.` wildcard is
  allowed).
- `max_age`: how long senders keep the policy, in seconds. A week (`604800`) or more for
  `enforce`.

3. Open the Worker's `….workers.dev` address with `/.well-known/mta-sts.txt` added: the four
   policy lines show.

## The hostname

On the Worker: the **Domains** tab, **Add Domain** (not **Add Route**).

- The dialog is a **two-step picker**: type the zone's name (`mailtransporter.co.uk`),
  select it, then enter the subdomain (`mta-sts.mail`). Typing the full hostname into the
  search box answers "No zones match", and **Onboard domain** would try to add the domain
  to Cloudflare again: cancel instead.
- Cloudflare creates the DNS record and orders a certificate for the hostname. The free
  certificate covers only one level of subdomain (`*.mailtransporter.co.uk`), so for a
  deeper name like this one it orders an **Advanced** certificate, at no cost. It shows
  under **SSL/TLS, Edge Certificates** as **Pending Validation (TXT)**, then **Active**
  within minutes; the zone being on Cloudflare means nothing has to be added for the
  validation.
- A redirect cannot stand in for the Worker: sending servers don't follow redirects when
  they fetch the policy.

Check (expects `200` and the policy, then the certificate naming the hostname):

```bash
HOST=mta-sts.mail.mailtransporter.co.uk
curl -s -w "%{http_code}\n" "https://$HOST/.well-known/mta-sts.txt"
echo | openssl s_client -connect "$HOST:443" -servername "$HOST" 2>/dev/null | openssl x509 -noout -ext subjectAltName
```

If only your own computer gets `unrecognized name` or a certificate error, it is still
resolving the old server's address; see [Troubleshooting](troubleshooting.md).

## Reports without an inbox

TLS reports and Domain-based Message Authentication, Reporting and Conformance (DMARC)
reports arrive daily, as machine-written attachments, from every large mail provider. Nobody
needs to read them routinely: they matter before tightening a policy and when tracking down
a delivery problem. So keep them out of any inbox and let SES store them.

The report address **cannot be on the sending subdomain**: its `MX` must point at SES's
bounce endpoint (`feedback-smtp…`), which processes bounces and discards everything else.
Use a subdomain of its own:

| Type | Name | Content | Proxy |
|---|---|---|---|
| `MX` | `reports` | `inbound-smtp.eu-west-1.amazonaws.com` (priority `10`) | DNS only |

- SES receiving, in the same region as the identity (**Email receiving**, active rule set):
  a rule with no recipient conditions accepts mail for every verified domain in that
  region, so the new subdomain needs no change there. Its action decides where mail goes;
  here it is **Deliver to Amazon S3 bucket**, into the Amazon Simple Storage Service (S3)
  bucket `mailtransporter`.
- A lifecycle rule on the bucket (**Management, Lifecycle rules**) deletes objects after 7
  days, so reports never pile up.
- Both report records point at the same address: `rua=` and `ruf=` in the sending
  subdomain's `_dmarc` record, and `rua=` in `_smtp._tls`. DMARC reports to an address on
  **another** domain also need a permission record on that domain; on the same domain they
  don't.

Check: send any email to `report@reports.mailtransporter.co.uk`; it appears as an object in
the bucket within a minute.

---

Previous: [Domains that only redirect](redirect-only-domains.md) | Next: [Checks](checks.md)
