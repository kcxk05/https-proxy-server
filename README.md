# https proxy server: How It Works, How to Set One Up in Chrome, cURL, and Python, and When to Stop Relying on Free Lists

People searching for an HTTPS proxy server usually want one of two things. Either they want to understand what "HTTPS" adds to a proxy before they buy anything, or they already have an `ip:port` string and need to know where to paste it.

Both are covered here. The short version first: an HTTPS proxy is not a magic privacy upgrade, and the free lists you find on page one are mostly dead endpoints operated by strangers. If you need something that still works tomorrow, you're shopping for a paid residential network. That's where 9Proxy comes in later on, but the mechanics come first.

## What "HTTPS proxy server" actually means

A proxy is just an intermediary. Your client opens a connection to the proxy, tells it where to go, and the proxy relays the traffic. What changes between an HTTP proxy and an "HTTPS proxy" is how it handles encrypted destinations.

For a plain HTTP request, the proxy reads the request line, forwards it, and can cache or rewrite the response. For an HTTPS destination, that option doesn't exist, because the payload is encrypted. Instead, the client sends a `CONNECT` request asking the proxy to open a raw TCP tunnel to `host:443`. The proxy either allows it or refuses it. If it allows, the TLS handshake happens end to end between you and the site, and the proxy only forwards bytes it can't read.

Two things follow from that, and both get misreported constantly.

**The proxy still knows where you're going.** Even with a CONNECT tunnel, the proxy sees the destination hostname and IP, the ports, the timing, and the volume. It doesn't see your password or the page content, but traffic analysis doesn't need those. "HTTPS encrypted" and "the proxy can't see anything" are different statements.

**"HTTPS proxy" is a capability, not a port.** Some tools let you write `https://proxy.example:8080`, which usually means the connection *to the proxy* is wrapped in TLS, not that the proxy decrypts your traffic. In cURL, `-x http://host:port` is the normal way to reach an HTTPS target, and it uses CONNECT under the hood. If you point `-x` at an `https://` URL, cURL assumes the proxy itself speaks TLS on that port. These are separate features and providers blur them in marketing copy.

### HTTP proxy vs. HTTPS-capable proxy vs. SOCKS5

| Type | Handles HTTPS targets | Sees content | Works for non-HTTP traffic | Notes |
| --- | --- | --- | --- | --- |
| HTTP-only proxy | Often no (CONNECT blocked) | Yes, for plain HTTP | No | Fine for caching, poor for modern sites |
| HTTP proxy with CONNECT | Yes | No, sees metadata only | No | The standard "HTTPS proxy" setup |
| SOCKS5 | Yes | No, sees metadata only | Yes (any TCP) | No HTTP header rewriting, so it looks more like a normal client |
| Transparent proxy | Yes | Yes (often adds `X-Forwarded-For`) | Varies | Usually an ISP or captive-portal thing, not something you choose |

There's also an anonymity classification you'll see on proxy-list sites: transparent, anonymous, and elite. Transparent proxies leak your real IP in headers. Anonymous ones hide it but announce themselves as proxies. Elite ones send no telltale headers at all, though the target can still probe the IP and figure out it's a proxy.

## Four ways to get one, with the trade-offs attached

### 1. Free public proxy lists

This is where most searches end up, so it deserves a straight answer rather than a lecture. The format is always the same: a page or a GitHub repo publishes `ip:port` lines, checked by a bot, refreshed periodically.

The numbers are worse than the free-list sites imply. A long-running academic measurement study tested roughly 180,000 proxies over ten months and found that fewer than 2% were actually proxying traffic, with about 10% injecting ads or attempting to intercept TLS connections. Another analysis of open proxies found a median lifespan of about seven days.

There's also a structural problem specific to HTTPS. The `proxy-free/free-proxy-list` repo publishes separate files for HTTP, SOCKS4, and SOCKS5, and its HTTPS file sits at roughly one working entry out of ~2,670 proxies, because most public proxies simply can't tunnel HTTPS through `CONNECT`. The repo maintainers say so plainly: if you need to fetch HTTPS URLs, use the SOCKS5 list instead.

Free proxies are usable for a single unauthenticated page view or for learning where the setting lives in your OS. They are not usable for anything you'd be annoyed to lose.

### 2. Self-hosted on a VPS

Run Squid for HTTP/HTTPS or Dante for SOCKS5 on a cheap VPS and you get a private, reliable endpoint for a few dollars a month. The catch is that the IP belongs to a hosting provider, so it's a datacenter IP. Sites that care about this will classify it by ASN in milliseconds. Self-hosting solves reliability and operator trust, not block rates.

### 3. Regional residential proxy services

If the job is "this request needs to look like it came from a real home connection in a specific city," you're paying for residential IPs. These are addresses assigned by ISPs to real devices, which is why they clear anti-bot checks that datacenter ranges fail instantly.

The billing model matters more than the logo on the homepage. You'll see per-IP pricing with unlimited bandwidth, per-GB pricing with rotating endpoints, or bundles that mix both.

### 4. Browser extensions and desktop apps

Convenient, and generally a thinner wrapper around one of the above. Read the permissions before installing. Some free "proxy" extensions fund themselves by routing *other people's* traffic through your connection, which usually isn't obvious from the store listing.

## How to configure one in practice

### Windows and Chrome

Chrome on Windows follows the system proxy. `Settings → Network & internet → Proxy` lets you set a manual HTTP/HTTPS proxy and a bypass list. Chrome on macOS does the same through System Settings. On Linux desktops, `gsettings set org.gnome.system.proxy mode 'manual'` plus the host/port keys works, and `http_proxy`/`https_proxy` environment variables cover most CLI tools.

One quirk worth knowing: Chrome's "Secure DNS" and WebRTC can leak your real IP even when the proxy is set correctly. Test after configuring, not before.

### Firefox

Firefox has its own proxy settings, independent of the OS. `Settings → Network Settings → Manual proxy configuration` includes SOCKS5 support and a "Proxy DNS when using SOCKS5" checkbox. Turn that on if you care about DNS leaks, because otherwise your resolver still sees every hostname.

### cURL and the command line

bash
# HTTP proxy reaching an HTTPS target (uses CONNECT internally)
curl -x http://USER:PASS@proxy.example.com:8080 https://api.ipify.org

# SOCKS5
curl --socks5-hostname USER:PASS@proxy.example.com:1080 https://api.ipify.org

# Confirm what the outside world sees
curl -s https://api.ipify.org


`--socks5-hostname` resolves DNS through the proxy. Plain `--socks5` resolves locally, which is the leak most people never notice.

### Python

python
import requests

proxy_url = "http://USER:PASS@proxy.example.com:8080"
proxies = {"http": proxy_url, "https": proxy_url}

r = requests.get("https://api.ipify.org", proxies=proxies, timeout=15)
print(r.text)


The most common mistake here is putting an `https://` scheme in the URL for the `https` key. For a standard HTTP proxy that tunnels HTTPS, the scheme stays `http://`. If the SDK rejects it or hangs, that's usually why. Rotating sessions are handled the same way rotating endpoints are: build a list, pick one per request, and handle timeouts rather than assuming every node is alive.

## Testing whether the tunnel actually works

Before you build retry logic around a proxy, confirm three things:

1. **The exit IP changed.** Hit an IP-echo endpoint and compare it to your real address.
2. **No DNS leak.** If the resolved IP of a hostname still routes through your local resolver, the proxy only covers part of the connection.
3. **No header leak.** Some proxies add `Via` or `X-Forwarded-For`. If those appear, the endpoint is anonymous at best, not elite.

Then, if the error message looks like this:


SSL certificate problem: self signed certificate in certificate chain


stop and think. On a properly configured CONNECT proxy, the certificate you see is the destination site's own. A substituted or self-signed chain is the classic signature of a TLS-terminating middleman. That's a reason to abandon the endpoint, not to add `--insecure` and move on.

## What to check before paying for HTTPS proxies

Once you've decided the free tier has run its course, the checklist is short and mostly boring:

- **IP type.** Residential or static ISP addresses for sites with real bot detection. Datacenter ranges are cheap and get flagged fast.
- **Rotation control.** Rotating sessions for high-volume unrelated requests, sticky sessions when an account needs to stay on one address long enough to log in and browse.
- **Geo-targeting depth.** Country-level is table stakes. City or ZIP-level is a different product.
- **Authentication options.** Username/password for cloud tools; IP whitelisting for servers where you don't want credentials in a config.
- **Expiry terms.** Some GB packages expire in 30 days, which is a problem if your usage is spiky. Anything that expires slowly is worth more than a slightly lower per-GB rate.
- **Replacement policy.** Residential IPs die. How fast the provider swaps a dead one is the difference between a script that finishes and one that doesn't.

## How 9Proxy maps onto that list

9Proxy runs a residential network of 20M+ IPs across 90+ countries, and it offers two billing models that answer two different questions.

**Residential Proxy by IPs** is for workloads where the address has to stay put. You buy a fixed number of residential IPs, bandwidth on those IPs is unlimited, and unused IPs never expire. Geo-targeting covers country, state, city, ZIP, and ISP. The trade-off is setup: this model works through the 9Proxy desktop app, which does local port forwarding, and the IPs themselves are genuinely residential, so they last anywhere from a few hours to about 24 hours.

**Residential Proxy by GB** flips it. You buy a volume of traffic, generate unlimited endpoints from the dashboard, and pay for what you consume. No app required, since everything runs in the browser dashboard. You authenticate with username/password or IP whitelisting, and you can pick sticky sessions (hold the same IP for X minutes) or rotating (new IP per request or session). Traffic is valid for 180 days.

Both models support HTTP(S) and SOCKS5, which is exactly the pairing you need if you're mixing browser work with scripts.

The operational features matter more than the headline pool size once you're actually running jobs. Every proxy that dies within 60 seconds is replaced automatically. Anything you used in the last 24 hours shows up in a "Today List" you can reuse at no extra cost, which quietly cuts spend on daily repeat tasks like rank tracking. Auto-rotation and auto-refresh handle the manual babysitting, and share codes or sub-accounts let you hand access to a teammate without passing around your login.

Independent reviews are mixed but consistent on the fundamentals. Geekflare's review measured a 97.7% success rate against Cloudflare-protected targets and called the dual billing model the real advantage versus enterprise-priced competitors. ProxyLook rates the service around 3.9 out of 5. One reviewer measured roughly 99.5% success and about 0.6s average response time on their own tests while noting that residential IPs still drop, which is true of every provider in this category.

The honest limitations: the desktop app requirement for IP-based packages is friction on multi-device setups, streaming services like Netflix are a weak spot because residential IPs get profiled there aggressively, and the pool is smaller than Bright Data's, so highly specialized targeting may run thin. If your work is enterprise-scale compliance scraping, you may need more tooling than this.

## 9Proxy pricing: every package currently listed

Prices below reflect the adjustment 9Proxy announced on May 18, 2026. Worth knowing before you compare numbers on other review sites:

> On June 1, 2026, 9Proxy raised prices for IP-based and bundle packages. GB-based package pricing was not changed. Older reviews and coupon pages still show the previous, lower IP-based figures, so a 100-IP quote of $20 circulating online is out of date.

### IP-based residential packages

One-off purchase, unlimited bandwidth, IPs never expire.

| Package | IPs | Price | Effective per IP | Buy |
| --- | --- | --- | --- | --- |
| Entry | 100 | $24 | $0.24 | [Check the 100 IP package](https://9proxy.com/pricing?inviteCode=9P_VIPOFF20) |
| Small | 500 | $72 | $0.144 | [Check the 500 IP package](https://9proxy.com/pricing?inviteCode=9P_VIPOFF20) |
| Value | 1,000 + 500 bonus | $126 | $0.084 | [Check the 1,500 IP package](https://9proxy.com/pricing?inviteCode=9P_VIPOFF20) |
| Standard | 2,500 | $210 | $0.084 | [Check the 2,500 IP package](https://9proxy.com/pricing?inviteCode=9P_VIPOFF20) |
| Mid | 5,000 | $360 | $0.072 | [Check the 5,000 IP package](https://9proxy.com/pricing?inviteCode=9P_VIPOFF20) |
| Large | 15,000 | $720 | $0.048 | [Check the 15,000 IP package](https://9proxy.com/pricing?inviteCode=9P_VIPOFF20) |
| Large+ | 25,000 | $863 | $0.035 | [Check the 25,000 IP package](https://9proxy.com/pricing?inviteCode=9P_VIPOFF20) |
| Bulk | 50,000 | $1,438 | $0.029 | [Check the 50,000 IP package](https://9proxy.com/pricing?inviteCode=9P_VIPOFF20) |
| Business | 100,000 | $2,300 | $0.023 | [Check Business IP pricing](https://9proxy.com/pricing?inviteCode=9P_VIPOFF20) |
| Business | 200,000 | $4,140 | $0.021 | [Check Business IP pricing](https://9proxy.com/pricing?inviteCode=9P_VIPOFF20) |
| Business | 500,000 | $8,625 | $0.018 | [Check Business IP pricing](https://9proxy.com/pricing?inviteCode=9P_VIPOFF20) |

### GB-based residential packages

Pay for traffic, generate unlimited endpoints. All tiers carry 180-day validity, unlimited on Enterprise.

| Package | Price | Effective per GB | Buy |
| --- | --- | --- | --- |
| 5 GB | $15 | $3.00 | [Check the 5 GB pack](https://9proxy.com/pricing?inviteCode=9P_VIPOFF20) |
| 50 GB + 5 GB bonus | $105 | $2.10 | [Check the 55 GB pack](https://9proxy.com/pricing?inviteCode=9P_VIPOFF20) |
| 100 GB | $150 | $1.50 | [Check the 100 GB pack](https://9proxy.com/pricing?inviteCode=9P_VIPOFF20) |
| 200 GB | $200 | $1.00 | [Check the 200 GB pack](https://9proxy.com/pricing?inviteCode=9P_VIPOFF20) |
| 1,000 GB | $800 | $0.80 | [Check the 1,000 GB pack](https://9proxy.com/pricing?inviteCode=9P_VIPOFF20) |
| 2,000 GB | $1,500 | $0.75 | [Check the 2,000 GB pack](https://9proxy.com/pricing?inviteCode=9P_VIPOFF20) |
| Enterprise GB | Custom | Down to $0.68 at the highest tier | [Request Enterprise pricing](https://9proxy.com/pricing?inviteCode=9P_VIPOFF20) |

### Bundle packages (IPs + traffic together)

| Bundle | Contents | Price | Buy |
| --- | --- | --- | --- |
| Starter | 100 IPs + 5 GB | $30 | [Check the Starter bundle](https://9proxy.com/pricing?inviteCode=9P_VIPOFF20) |
| Popular | 1,500 IPs + 50 GB | $180 | [Check the Popular bundle](https://9proxy.com/pricing?inviteCode=9P_VIPOFF20) |
| Pro | 5,000 IPs + 500 GB | $720 | [Check the Pro bundle](https://9proxy.com/pricing?inviteCode=9P_VIPOFF20) |

Enterprise also changes the account structure rather than just the price: one owner plus up to five members, bandwidth shared inside the team without expiry, per-member traffic limits, activity logs, unlimited share codes, and dedicated support.

## Picking a package without overthinking it

**Just testing whether residential IPs fix your block rate:** the 5 GB pack at $15, or the 100 IP package at $24 if your requests are session-based. Both are small enough that being wrong costs less than lunch.

**Daily SEO rank tracking or competitor price monitoring:** GB-based, starting at the 50 GB + 5 GB tier. These jobs rotate aggressively and burn bandwidth without needing persistent addresses. The 180-day validity means a light month doesn't waste the balance.

**Multi-account work where each account needs its own stable address:** IP-based. Start at 100 IPs and scale in the same tiers. The "never expire" term means unused IPs aren't burning a clock while you work through them.

**Mixed workloads or client-agency work:** the Starter or Popular bundle. Starter at $30 is cheaper than buying 100 IPs ($24) and 5 GB ($15) separately, and it keeps one project's budget in one line.

If the decision is close, the tiebreaker is expiry, not price. A GB tier that costs 6% more but lasts six months beats a 30-day pack you'll half-waste.

## Discounts, trials, and how to get in cheaper

9Proxy's affiliate program gives referred users 5% off their purchases, which the invite link takes care of automatically.

👉 [Grab the 5% invite discount on 9Proxy](https://9proxy.com/pricing?inviteCode=9P_VIPOFF20)

Trials exist but are limited and granted through support rather than a self-serve button, and reviewers note they're promotion-dependent. If you want to test before paying, ask support whether they have IP-based or GB-based trial capacity at the moment, and be specific about which one you need.

The provider also runs periodic promotions rather than one permanent coupon. Green Sunday, for example, credited +10% back on Sunday IP and GB usage, capped at 1,000 IPs and 100 GB per week, with the bonus landing Monday. Offers like that rotate, so check what's live on the account dashboard instead of trusting a coupon aggregator page, which is usually republishing something expired.

## Questions that come up often

**Is an HTTPS proxy the same as a VPN?**
No. A VPN tunnels all traffic from the device at the network layer. A proxy applies to whatever you point at it, which can be a single browser profile or a single script. A VPN hides your IP from your ISP; a proxy hides it from the target site.

**Do I need HTTPS proxies or SOCKS5 proxies?**
If you're only fetching HTTPS URLs through a supported client, an HTTP proxy with CONNECT is enough. Choose SOCKS5 when you need non-HTTP traffic, UDP-adjacent protocols, or when you want to avoid HTTP header rewriting. 9Proxy supports both, so it's a config choice rather than a purchase decision.

**Why do my requests still get blocked with a residential IP?**
Usually because of behavior, not the IP. Identical headers, no cookies, machine-gun timing, and indefinite reuse of one address all look automated regardless of where the IP lives. Rotating sessions and realistic request pacing fix more blocks than upgrading your provider does.

**Can I use one proxy for everything?**
You can, and you'll regret it. Split by job: sticky sessions for anything with a login, rotating for bulk reads, and a separate address for each account you care about keeping.

**Will this work on my phone?**
GB-based plans work from the dashboard with username/password or whitelisted IP, which covers mobile-side tools. IP-based plans route through the desktop app, so legacy-IP workflows stay on a computer.

## The short version

An HTTPS proxy server is a CONNECT tunnel plus a relay that can't read your payload but can still see your destination. Setting one up takes about five minutes in a browser or three lines of code. The hard part was never the configuration, it's finding an endpoint that still exists next week and doesn't belong to someone with a side business in injected ads.

If you're past the experimenting stage, the useful decision is IP-based or GB-based, and the answer comes straight from your workload: persistent addresses for accounts and sessions, metered traffic for volume and rotation.

👉 [Compare all 9Proxy plans and start with the smallest fit](https://9proxy.com/pricing?inviteCode=9P_VIPOFF20)
