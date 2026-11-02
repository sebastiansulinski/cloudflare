[Contents](../README.md)

# Move a domain's DNS to Cloudflare

The domain's Domain Name System (DNS) records move to Cloudflare first, with the site or
redirect ready before the nameservers change, so nothing goes offline.

## Record the current records

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
signing records ending in `._domainkey`: no lookup can list them. Their names are in the
Amazon Simple Email Service console (**Identities**, the domain, **DomainKeys Identified Mail
(DKIM), Publish DNS records**).

## Switch off DNSSEC at the old registrar

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

## Add the zone

1. Cloudflare, account home: **Add a domain** (the button may read **Onboard a domain** or
   **Connect a domain**).
2. Enter the domain, let it scan for existing records, choose the **Free** plan.
3. Note the two nameservers Cloudflare shows (`<name>.ns.cloudflare.com`). Don't switch yet.

The zone shows **Pending** until the nameservers change. Records, routes, rules and
settings can all be set up while it is pending.

## Check the imported records

**Domain, DNS, Records.** Compare with the list from [Record the current records](#record-the-current-records).
The import guesses names, so it **silently misses** anything under a name it doesn't know:
on `mailtransporter.co.uk` it dropped all three Amazon Simple Email Service signing records,
`_mta-sts.mail` and `_smtp._tls.mail`, while finding `_dmarc.mail` beside them. Outgoing mail
lost its signatures the moment the nameservers changed.

Ask the old and new nameservers for the same names side by side (any line printed is a
record that differs). The old provider keeps answering for days after the switch, so this
works afterwards too:

```bash
DOMAIN=example.co.uk
OLD=dns1.registrar-servers.com
NEW=glen.ns.cloudflare.com
for n in @ www mail _dmarc _dmarc.mail _mta-sts.mail _smtp._tls.mail google._domainkey; do
  [ "$n" = @ ] && f=$DOMAIN || f=$n.$DOMAIN
  for t in A CNAME MX TXT; do
    a=$(dig +short $t "$f" @$OLD | sort | tr '\n' ' '); b=$(dig +short $t "$f" @$NEW | sort | tr '\n' ' ')
    [ -n "$a$b" ] && [ "$a" != "$b" ] && printf "%-6s %-40s old: %s\n%-47s new: %s\n" $t "$f" "$a" "" "$b"
  done
done
```

Add the domain's own names to the `for` list, including each `<key>._domainkey` from the Amazon
Simple Email Service console. `NEW` is either of the zone's two Cloudflare nameservers
([Add the zone](#add-the-zone)). Nothing printed means the two agree.

Then go through:

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
  domain and `www` (the tunnel routes replace them, [Serve a site through the tunnel](serve-site-through-tunnel.md)).
  For a domain that only redirects, delete everything and follow
  [Domains that only redirect](redirect-only-domains.md).
- **Wildcards:** delete an imported `*` record unless something really uses it.

Apply the [zone settings](zone-settings.md) now too.

## Switch the nameservers

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
certificate as they are until the overlap ends ([Certificates and closing the public ports](nginx.md#certificates-and-closing-the-public-ports)).

---

Previous: [Before you start](before-you-start.md) | Next: [Transfer the registration](transfer-registration.md)
