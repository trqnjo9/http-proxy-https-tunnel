# http vs https proxy: what the two actually do to your traffic, and which one to point your scraper at

Most people who land on this question are stuck on something specific. Either they configured a proxy as `http://user:pass@host:port` and now want to know whether that's still safe when the target is an `https://` site. Or they bought a plan sold as "HTTPS proxies" and can't work out how it differs from the HTTP one next to it. Or their scraper cruises through some targets and gets blocked on others, and they suspect the protocol is the reason.

Two of those are the same question about vocabulary. One is about the port you dial. None of them are really about which provider to buy — which is the part most write-ups bury under five paragraphs of glossary.

So let's take the terminology apart first, because almost every "http vs https proxy" argument online is two people using the same phrase for different things.

## What a proxy URL says, and what it doesn't

There are two independent properties, and they get fused into one word constantly.

**Property one: how you talk to the proxy server.** That's the scheme in your proxy address.

- `http://proxy.example.com:8080` — you speak plain HTTP to the proxy. Anyone sitting on the path between you and the proxy can read that leg of the conversation: the target hostname, your proxy credentials, everything.
- `https://proxy.example.com:8080` — you establish a TLS connection to the proxy first, then talk over it. The leg between you and the proxy is encrypted.

**Property two: what happens to the traffic once it's inside.** That depends on the destination, not on your proxy's scheme:

- Destination is `http://` — the proxy receives a readable HTTP request. It can see the URL, headers and body, and it can read, modify or cache the response.
- Destination is `https://` — your client sends a `CONNECT host:443` line to the proxy, the proxy opens a raw TCP connection to that host, replies `200 Connection Established`, and from then on it's a byte relay. The TLS handshake runs between your client and the destination server. The proxy doesn't have the destination's private key, so it can't decrypt anything.

Here's the bit that trips people up: if your target is `https://`, you are using an HTTPS proxy in the meaningful sense **even if your proxy address starts with `http://`**. The `http://` prefix only tells you that the leg to the proxy is unencrypted. It says nothing about whether the tunnel is encrypted.

Turn it around and the same logic holds. A proxy address with an `https://` prefix pointing at a plain `http://` destination is just a proxy doing ordinary plaintext HTTP work. The prefix bought you encryption on one hop, not on the transaction.

> If you remember one rule: whether you're running an HTTPS proxy is decided by the target URL, and the proxy prefix only decides whether the hop to the proxy is encrypted.

## What an HTTP proxy does with your request

An HTTP proxy operates at the application layer. It understands HTTP semantics — request methods, headers, status codes — which is exactly why it's useful for scraping and automation and exactly why it's a privacy problem for anything sensitive.

When a plaintext HTTP request goes through it, the proxy sees the full request line and all headers: your User-Agent, `Accept-Language`, cookies, referrer, and the body. It can rewrite any of it, drop a header, inject an `X-Forwarded-For`, or serve a cached response without ever touching the origin server.

That visibility is not a bug. Content filtering gateways rely on it to block URLs by pattern. Corporate web application firewalls use it to inspect requests before they hit internal apps. CDN edge nodes use it to avoid re-fetching. And scrapers get real value from it, because being able to rotate User-Agent and Accept-Language per request is often the difference between a 200 and a 403.

The tradeoff: anything the proxy can see, whoever runs the proxy can see. On a plaintext HTTP destination, "the proxy is a blind relay" is simply false.

## What an HTTPS proxy actually does: the CONNECT tunnel

Once your target is HTTPS, the proxy's role changes from inspector to pipe.

Your client opens a connection and sends something like:


CONNECT github.com:443 HTTP/1.1
Host: github.com:443


The proxy resolves and connects to `github.com:443` over TCP, responds `200 Connection Established`, and then copies bytes in both directions without parsing them. The TLS handshake that follows is between your client and GitHub's server. Since the proxy never has GitHub's private key, there's nothing for it to decrypt.

For practical purposes, your traffic to the destination is end-to-end encrypted even through a proxy you don't trust much. That's the real security win of the CONNECT approach.

It is not invisibility, though. Two pieces of metadata stay exposed:

1. **The CONNECT line itself is plain text.** Anyone observing your link to the proxy — your ISP, a network operator, a logging layer — can see you're connecting to `github.com:443`. They can't see what you did there.
2. **SNI in the TLS ClientHello** carries the destination hostname in the clear. Even inside an encrypted handshake, the hostname is visible unless the connection uses Encrypted Client Hello.

If your threat model includes hiding *which sites you visit* rather than just *what you did on them*, a proxy is the wrong tool. Traffic through an HTTPS proxy leaks destination hostnames by design. A VPN encrypts the metadata too, which is the one thing a CONNECT tunnel doesn't do.

There's also a third thing bearing the "HTTPS proxy" name that isn't a proxy in this sense at all: **TLS interception**. Corporate and some government deployments install their own root certificate on your device and terminate TLS at the gateway, re-encrypting onward. That's SSL inspection, and it does read your traffic. If your machine trusts an unexpected root CA, that's what's happening — not a normal HTTPS proxy.

## HTTP proxy vs HTTPS proxy, side by side

|  | HTTP proxy (plaintext destination) | HTTPS proxy (CONNECT tunnel) |
| --- | --- | --- |
| Layer | Application (L7) | Application + CONNECT tunnel |
| What the proxy reads | Full request: URL, headers, body | Nothing inside the tunnel |
| Can modify headers | Yes | No |
| Can cache responses | Yes | No — it never sees the response |
| Encryption to destination | None | TLS, end to end |
| Metadata visible to observers | Almost everything | Destination hostname via CONNECT and SNI |
| Typical port | 80 or 8080 | 443 |
| Protocol support | HTTP/HTTPS web traffic only | Anything you can tunnel over TCP |
| Overhead | Header injection, roughly a few hundred bytes per request | Extra TLS handshake before first byte |

The most useful line in that table for most readers is the caching one. If you're hammering the same target repeatedly, an HTTP proxy with connection reuse and caching can be genuinely faster and cheaper. If you're pushing login credentials or session cookies, the tunnel is not optional.

## Do you actually need a separate HTTPS port?

Usually no, and this is where a lot of people waste money or configure things twice.

Python's `requests` and Node's `axios` behave the same way: you hand them an `http://` proxy endpoint and they route HTTPS destinations through it automatically using CONNECT. The client does the work. You don't need a second, TLS-wrapped proxy address.

A dedicated HTTPS endpoint makes sense when:

- your client or network stack requires it explicitly (some JVM setups, some enterprise middleware),
- you're on an untrusted network and want the leg to the proxy encrypted too,
- or you're in a jurisdiction or corporate environment where the proxy hop itself being readable is a problem.

Some providers split the two across separate ports precisely for this — Zenrows, for example, exposes HTTP on one port and HTTPS on another, and its own documentation notes that most HTTP clients only need the HTTP port. Others hand you one endpoint that handles both. Either way, the decision is about your client, not about which product tier you're on.

## What the extra encryption hop costs

TLS to the proxy isn't free. You pay one extra round trip before the first byte of response arrives, then whatever symmetric crypto overhead applies to the session.

Vendor benchmarks on this vary a lot and almost all of them are published by someone selling proxies, so treat the numbers as directional rather than gospel. One published comparison from VoidMob put HTTPS proxy handshakes in the 90–160 ms range against roughly 45 ms for HTTP, and flagged the double encryption — client to proxy, then client to destination — as the cause. A Chinese-language benchmark done on the same residential nodes against a North American e-commerce target found HTTPS slowest on first connection (roughly 2.4× HTTP) but within about 15% once connections were reused.

The practical read: on long-running sessions with connection pooling, HTTp-versus-HTTPS-to-the-proxy stops mattering. On thousands of short-lived connections, it adds up. And SOCKS5, which works at a lower layer and doesn't parse anything, has the smallest per-request overhead of the three — which is why high-volume automation often ends up there instead.

## Matching the protocol to the job

| Workload | Best fit | Why |
| --- | --- | --- |
| High-volume scraping of the same targets | HTTP proxy | Header control, connection reuse, caching |
| Login flows, carts, account sessions | HTTPS (CONNECT) | Credentials never visible to the proxy |
| Payment or API keys in transit | HTTPS (CONNECT) | End-to-end encryption to the destination |
| Mixed protocols, non-HTTP traffic | SOCKS5 | Protocol-agnostic TCP/UDP relay |
| Browser automation (Playwright, Puppeteer, Selenium) | HTTP or SOCKS5 | Native support in all three, no extra config |
| Debugging what a client is actually sending | HTTP proxy | Only mode where you can read and edit the request |

The reason this table matters to a buying decision is that the two things people conflate — proxy type and proxy vendor — are different axes. Your vendor needs to support both protocols on the same IP pool, so switching per workload doesn't mean buying a second product.

9Proxy, for instance, lists HTTP/HTTPS and SOCKS5 support across the same residential network — 20M+ residential IPs across 90+ countries, with targeting down to country, state, city, ZIP and ISP. That means the protocol question stays a configuration detail rather than a procurement one. 👉 [Check 9Proxy's residential proxy options](https://bit.ly/9-Proxy) if you want both under one account.

## The part that actually changes your bill

Once you've settled on protocol, the next question is how 9Proxy charges for it — and here the company genuinely gives you two different mental models.

**Residential by IPs.** You buy a fixed number of residential IPs. Bandwidth on them is unlimited while they're active, and unused IPs never expire. Each IP stays alive anywhere from a few hours to about 24 hours. It's the model for sustained sessions, heavy data transfer, and workflows where you can't predict bandwidth but you do know you need persistent addresses.

Worth knowing: IP-based plans require the 9Proxy desktop app for local port forwarding (with optional proxy authentication). Auto-rotation on selected ports is supported if you want to cycle addresses on a schedule.

**Residential by GB.** You buy traffic and generate as many endpoints as you like. Sessions can be sticky or rotating, per request or per configured interval, and authentication is either username/password or IP whitelisting. This runs entirely from the dashboard — no app. It's the model for high-rotation work where each request is small: light scraping, geo-checking, ad verification, API polling. All GB plans carry 180-day validity; the Enterprise tiers remove the expiry entirely.

Both models are prepaid balance, not subscriptions. You top up and spend.

### Current 9Proxy plans and pricing

These are 9Proxy's published rates in USD following its June 1, 2026 pricing adjustment, which raised IP-based and Bundle prices while leaving GB-based prices unchanged. All are one-time balance purchases. Worth confirming the exact figure at checkout, since package contents and prices have moved more than once.

| Package | What you get | Price | Validity | Get it |
| --- | --- | --- | --- | --- |
| 100 IPs | Unlimited bandwidth per IP | $24 | IPs never expire | [ Buy 100 IPs](https://bit.ly/9-Proxy) |
| 500 IPs | Unlimited bandwidth per IP | $72 | IPs never expire | [ Buy 500 IPs](https://bit.ly/9-Proxy) |
| 1,000 IPs + 500 bonus | Unlimited bandwidth per IP | $126 | IPs never expire | [ Buy the 1,500 IP pack](https://bit.ly/9-Proxy) |
| 2,500 IPs | Unlimited bandwidth per IP | $210 | IPs never expire | [ Buy 2,500 IPs](https://bit.ly/9-Proxy) |
| 5,000 IPs | Unlimited bandwidth per IP | $360 | IPs never expire | [ Buy 5,000 IPs](https://bit.ly/9-Proxy) |
| 15,000 IPs | Unlimited bandwidth per IP | $720 | IPs never expire | [ Buy 15,000 IPs](https://bit.ly/9-Proxy) |
| 25,000 IPs | Unlimited bandwidth per IP | $863 | IPs never expire | [ Buy 25,000 IPs](https://bit.ly/9-Proxy) |
| 50,000 IPs | Unlimited bandwidth per IP | $1,438 | IPs never expire | [ Buy 50,000 IPs](https://bit.ly/9-Proxy) |
| 100,000 IPs (Business) | Unlimited bandwidth per IP | $2,300 | IPs never expire | [ Buy the 100,000 IP pack](https://bit.ly/9-Proxy) |
| 200,000 IPs (Business) | Unlimited bandwidth per IP | $4,140 | IPs never expire | [ Buy the 200,000 IP pack](https://bit.ly/9-Proxy) |
| 500,000 IPs (Business) | Unlimited bandwidth per IP | $8,625 | IPs never expire | [ Buy the 500,000 IP pack](https://bit.ly/9-Proxy) |
| 5 GB | Rotating or sticky endpoints, no app needed | $15 | 180 days | [ Buy 5 GB](https://bit.ly/9-Proxy) |
| 50 GB + 5 GB bonus | Rotating or sticky endpoints | $105 | 180 days | [ Buy the 55 GB pack](https://bit.ly/9-Proxy) |
| 100 GB | Rotating or sticky endpoints | $150 | 180 days | [ Buy 100 GB](https://bit.ly/9-Proxy) |
| 200 GB | Rotating or sticky endpoints | $200 | 180 days | [ Buy 200 GB](https://bit.ly/9-Proxy) |
| 1,000 GB | Rotating or sticky endpoints | $800 | 180 days | [ Buy 1,000 GB](https://bit.ly/9-Proxy) |
| 2,000 GB | Rotating or sticky endpoints | $1,500 | 180 days | [ Buy 2,000 GB](https://bit.ly/9-Proxy) |
| 3,000 GB (Enterprise) | Team mode, per-member traffic controls, logs | $2,160 | Unlimited | [ Get Enterprise 3,000 GB](https://bit.ly/9-Proxy) |
| 6,000 GB (Enterprise) | Team mode, per-member traffic controls, logs | $4,200 | Unlimited | [ Get Enterprise 6,000 GB](https://bit.ly/9-Proxy) |
| 10,000 GB (Enterprise) | Team mode, per-member traffic controls, logs | $6,800 | Unlimited | [ Get Enterprise 10,000 GB](https://bit.ly/9-Proxy) |
| Starter Bundle | 100 IPs + 5 GB | $30 | 180 days on the traffic | [ Buy the Starter Bundle](https://bit.ly/9-Proxy) |
| Popular Bundle | 1,500 IPs + 50 GB | $180 | 180 days on the traffic | [ Buy the Popular Bundle](https://bit.ly/9-Proxy) |
| Pro Bundle | 5,000 IPs + 500 GB | $720 | 180 days on the traffic | [ Buy the Pro Bundle](https://bit.ly/9-Proxy) |

Paying per IP versus per GB is the single biggest cost lever here, and it maps directly onto the HTTP-versus-HTTPS question you started with.

Run the numbers on your own traffic and the answer usually falls out. If your HTTPS scraping pulls a few hundred kilobytes per page but you're keeping the same session alive across dozens of requests, GB billing quietly drains — you're paying for bandwidth you could have had free. If each request is a fresh CONNECT to a different host with a small response, an IP pack can sit mostly idle while GB billing matches your actual spend.

The bundles exist for the mixed case: some targets need a stable address, others just need volume. Starter at 100 IPs plus 5 GB is enough to test both behaviours against your own targets before committing.

### A few setup details worth knowing before you buy

- **Authentication differs by model.** GB-based plans accept username/password or IP whitelisting. IP-based plans route through the desktop app with optional proxy authentication. If you're deploying to a cloud VM or a headless worker, GB-based is the simpler path — no app to install.
- **Rotation behaves differently.** IP-based addresses don't rotate naturally; you rotate through the Auto Rotation Proxy at intervals you set on chosen ports. GB-based endpoints rotate per request in rotating mode, or switch at the end of a configured session in sticky mode.
- **Targeting goes down to city, ZIP and ISP** on both models, which matters if you're geo-checking rather than just hiding an IP.
- **Enterprise adds team features** rather than more network: one owner plus up to five members, per-member traffic limits, activity logs, and no expiry on shared bandwidth.
- **Trials exist but are limited** and subject to availability, and you'll be asked whether you want an IP-based or GB-based trial.
- **Payment** covers cards, crypto (USDT, BTC, ETH and others), Alipay, Apple Pay and Google Pay.

You can start on any of these with the same invite link: 👉 [Create your 9Proxy account](https://bit.ly/9-Proxy).

## Questions people actually ask

**Is `http://user:pass@host:port` safe for HTTPS sites?**
Your traffic to the destination is encrypted either way, because CONNECT handles that. What's exposed is the link between you and the proxy: the hostname you're connecting to and, if you use credential auth, your proxy login. On a home connection that's usually acceptable. On shared or public networks, use an `https://` proxy endpoint instead.

**Can an HTTPS proxy see my URLs?**
It sees the hostname from the CONNECT request and from SNI — so it knows you visited `example.com`. It cannot see the path, query string, headers or body, because those are inside the encrypted stream. Claiming a proxy "sees nothing" is wrong; claiming it reads your pages is also wrong.

**Will switching to HTTPS slow my scraper down?**
Only on connection setup, and only if you're connecting to the proxy over TLS. If you keep the `http://` proxy endpoint and let the client tunnel HTTPS through it, you add no proxy-side TLS overhead at all. Reusing connections rather than opening one per request will do more for your throughput than any protocol choice.

**Do I need SOCKS5 instead?**
Only if you're moving non-HTTP traffic, UDP, or want the lowest per-request overhead. For ordinary web scraping and browser automation, HTTP and HTTPS are supported natively by Selenium, Playwright, Puppeteer, `requests` and `axios` — no proxy plugin workarounds required.

**Does 9Proxy give me both?**
Yes. The network supports HTTP/HTTPS and SOCKS5 on the same residential pool, so you're choosing between a billing model rather than between protocols. 👉 [Compare the current 9Proxy packages](https://bit.ly/9-Proxy) and pick the one that matches how your requests are shaped.

## The short version

Your proxy's `http://` or `https://` prefix decides whether the hop to the proxy is encrypted. Your target URL decides whether you're running an HTTP proxy or an HTTPS proxy. Those are separate facts, and once you hold them apart, most of the confusion in this topic evaporates.

For scraping and automation, the default that works almost everywhere is an HTTP proxy endpoint that tunnels HTTPS destinations via CONNECT. Reach for a TLS-wrapped proxy endpoint when the link to the proxy matters. Reach for SOCKS5 when you're moving something other than web traffic. Then match your billing to your request shape — steady sessions to IP packs, bursty small requests to GB packs — instead of guessing.
