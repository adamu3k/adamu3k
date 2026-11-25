# WiFi Hub: full blueprint

A prepaid Wi-Fi billing system for a home server. Customers connect to the Wi-Fi, sign in on a captive portal, top up a wallet, buy time or data, and the system cuts them off automatically when it runs out. It includes a customer portal, an admin dashboard, promotions, giveaways and customer support.

Working name **WiFi Hub**. The brand name shown to customers is a setting (`brandName`, default `"ZENAA WiFi"`).

---

## 0. How to use this document

**For the owner:**

1. Put this folder (`wifi-hub/`) on the machine where you run Claude Code, open a terminal in it and run `claude`.
2. Type:
   > Read CLAUDE.md and docs/BLUEPRINT.md. Build Phase 0 and Phase 1 from §21. Stop after Phase 1 and show me the acceptance checks.
3. Continue phase by phase: "Build Phase 2", and so on.
4. When the app works in mock mode, copy the project to the home server and follow §19 to set up the real network.

**For Claude Code:** this is the source of truth. The sections are:

| § | Topic |
|---|---|
| 1 | Product summary |
| 2 | Network topology (how traffic flows) |
| 3 | System architecture |
| 4 | Tech stack and folder layout |
| 5 | Core concepts |
| 6 | Gateway (internet on/off, counting, cut-off) |
| 7 | Database |
| 8 | Business rules for money, time and data |
| 9 | Payments: wallet, MoMo direct, Paystack, vouchers, cash |
| 10 | Discounts and promotions |
| 11 | Giveaways, free trial, referrals |
| 12 | Customer support |
| 13 | Customer portal pages |
| 14 | Admin dashboard pages |
| 15 | API |
| 16 | Background jobs |
| 17 | Security |
| 18 | Configuration (.env and settings) |
| 19 | Docker and host setup |
| 20 | Testing |
| 21 | Build phases and acceptance checks |
| 22 | Operations and troubleshooting |
| 23 | Legal notes |

---

## 1. Product summary

**Customers:**

- Join the Wi-Fi "ZENAA WiFi". The phone shows a "Sign in to network" page (captive portal).
- Create an account (name, phone, password), or log in.
- See their **wallet money (GHS)**, **time left**, **data left**, active passes, devices and history.
- **Top up the wallet** with:
  - **MoMo direct**: send to the owner's MoMo number with a reference code. A relay phone reads the SMS and credits automatically, with no gateway fee.
  - **Paystack**: card or mobile money through Paystack checkout.
  - **Voucher code or QR code**: printed or shared cards.
  - **Cash at the counter**: staff credit it in the admin dashboard.
- **Buy a plan** from the wallet: time (e.g. 1 hour), a validity pass (e.g. 24-hour unlimited) or data (e.g. 2 GB in 7 days).
- Press **Connect**. Internet works until the pass runs out, then it **cuts off automatically** and the phone sees the portal again ("Time's up, top up?").
- Use **Deals**: promo codes, happy hour prices, top-up bonuses, free daily trial, giveaways, referral rewards.
- Get **Support**: open a ticket, chat with staff, "I paid but wasn't credited", FAQ.

**Owner and staff:**

- Live view of who is online, with time and data left. Kick, pause or ban.
- Customers: search, credit or debit the wallet, give free time, reset password, block.
- Plans and prices, speed tiers, promotions, promo codes, giveaways, voucher batches (print and QR).
- MoMo inbox (waiting, unclaimed SMS, history), Paystack payments, cash shifts.
- Support tickets and live chat, FAQ, announcements.
- Reports: revenue per day and method, plans sold, active users, data used.
- Settings: brand, currency, MoMo numbers, Paystack keys, network, staff and roles.

---

## 2. Network topology

The home server must **sit between customers and the internet**, because the server is what lets traffic through or blocks it. The existing Huawei router remains the internet source. Two modes are supported, chosen in `.env` (`GATEWAY_MODE`).

### Mode A: two interfaces (recommended, cannot be bypassed)

```
Internet ── Huawei router (192.168.8.1 or similar; owner's private network)
                │ Ethernet
                ▼ WAN_IF (e.g. enp1s0, DHCP from Huawei)
         ┌──────────────────────────────┐
         │ HOME SERVER (Linux)          │
         │  nftables · dnsmasq · Docker │
         │  hostapd (if Wi-Fi adapter)  │
         └──────────────────────────────┘
                ▲ LAN_IF = 10.50.0.1/24
                │   either: wlan0 (USB Wi-Fi adapter in AP mode, via hostapd)
                │   or:     eth1/USB-Ethernet → any old router in "Access Point" mode
                ▼
          Customer phones (10.50.0.10 – 10.50.0.250)
```

- Customers only ever see `10.50.0.0/24`. The owner's own network behind the Huawei is unreachable from it (nftables drops forwarding from LAN to the Huawei's private range).
- The server is the customers' DHCP server, DNS server and gateway.

### Mode B: single interface ("gateway on a stick", no extra hardware, can be bypassed)

```
Internet ── Huawei (DHCP OFF, still broadcasts its Wi-Fi) ── customers + server on the same network
                                                   server = DHCP server + gateway (e.g. 192.168.8.2)
```

- The owner turns off DHCP on the Huawei. The server's dnsmasq hands out addresses with `router = server IP`, so customer traffic goes Huawei Wi-Fi → server → Huawei → internet.
- **Weakness:** a customer who manually sets the Huawei as their gateway bypasses the system. The setup script and admin dashboard show this warning. Use it for testing or until Mode A hardware is available.
- The owner's own devices are added to the **Always allowed devices** list (bypass by MAC) so they're never blocked.

### Addresses (defaults, configurable)

| Item | Mode A | Mode B |
|---|---|---|
| Customer subnet | 10.50.0.0/24 | Huawei subnet, e.g. 192.168.8.0/24 |
| Gateway / portal IP | 10.50.0.1 | server's static IP, e.g. 192.168.8.2 |
| DHCP range | 10.50.0.10–250, lease 1 h | .100–.250, lease 1 h |
| Portal URL | `http://10.50.0.1` and `http://wifi.hub` | `http://192.168.8.2` and `http://wifi.hub` |

---

## 3. System architecture

```
                               ┌──────────────────────── HOME SERVER ───────────────────────────────┐
                               │                                                                     │
 Customer phone ─ DHCP/DNS ──► │ dnsmasq (host)  ── adds walled-garden IPs to nft set ──┐            │
        │                      │                                                        ▼            │
        │ HTTP :80 (not yet    │ nftables table "wifihub" (kernel)                                    │
        │ allowed → redirected)│   allowed set (ip . mac, per-entry timeout)                          │
        └────────────────────► │   per-session byte counters · NAT · captive redirect                 │
                               │        ▲ nft / conntrack commands                                    │
                               │ ┌──────┴──────────────────────────────────────┐   ┌──────────────┐  │
                               │ │ app container (network_mode: host)          │──►│ postgres     │  │
                               │ │  Fastify API + portal + admin (static)      │   │ (container)  │  │
                               │ │  gateway driver (nftables | mock)           │   └──────────────┘  │
                               │ │  scheduler: tick 15 s, expiry, reconcile    │   ┌──────────────┐  │
                               │ │  SSE realtime to portal and admin           │   │ backup       │  │
                               │ └─────────────────────────────────────────────┘   └──────────────┘  │
                               │        ▲                ▲                         ┌──────────────┐  │
                               │        │                │ webhooks, remote use ◄──│ cloudflared  │  │
                               └────────┼────────────────┼─────────────────────────┴──────────────┴──┘
                                        │                │
                 MoMo relay phone ──────┘          Paystack (api.paystack.co)
               (forwards "Payment received" SMS,
                sends thank-you / OTP SMS)
```

**Why `network_mode: host`:** the app has to see each customer's **real IP address** (to look up their MAC and allow them) and has to run `nft` and `conntrack` against the host's network. Postgres stays on the Docker bridge, published only on `127.0.0.1:5432`.

**Who does what:**

| Component | Where | Job |
|---|---|---|
| hostapd | host (Mode A with USB Wi-Fi) | broadcasts the customer SSID |
| dnsmasq | host | DHCP and DNS for customers, `wifi.hub` → portal IP, fills the walled-garden nft set |
| nftables table `wifihub` | kernel, **owned by the app** | block by default, allow paid devices, count bytes, redirect to the portal |
| app | Docker, host network | everything else: API, portal, admin, jobs, gateway driver |
| postgres | Docker | data |
| backup | Docker | nightly `pg_dump`, keep 14 days |
| cloudflared | Docker, optional profile | public HTTPS address for Paystack webhooks and the portal from outside |

---

## 4. Tech stack and folder layout

This is the same stack as the owner's ZENAA GameHub app, so code and habits carry over.

- **Node.js 22**, TypeScript (strict), ESM.
- **Fastify 5**, **zod** validation, **@fastify/jwt** (customer and staff tokens), **@fastify/static**, **@fastify/rate-limit**, **@fastify/cookie**.
- **PostgreSQL 16** with **drizzle-orm** and drizzle-kit migrations. Tests use **@electric-sql/pglite** (in-process Postgres).
- **vitest** for tests.
- Web: **React 18 + Vite + TypeScript**, plain CSS with CSS variables (light/dark), no UI framework and no CDN. Two entry points: `portal` (customers, mobile-first, tiny) and `admin` (staff).
- Realtime: **Server-Sent Events** (`/api/portal/events`, `/api/admin/events`), not websockets.
- QR codes: `qrcode` package (server generates SVG/PNG).
- Passwords: Node `crypto.scrypt`. IDs: UUID v4.
- Gateway: `nft` and `conntrack` CLIs (installed in the app image) through a driver interface; `mock` driver for development and tests.

```
wifi-hub/
├─ CLAUDE.md
├─ DEVELOPMENT_LOG.md
├─ docs/
│  ├─ BLUEPRINT.md               ← this file
│  ├─ DECISIONS.md
│  ├─ SETUP.md                   ← owner-facing install guide (written in Phase 6)
│  └─ MOMO-RELAY.md              ← owner-facing MoMo phone guide (Phase 2)
├─ .env.example
├─ docker-compose.yml
├─ Dockerfile                    ← multi-stage: build web + server, run node
├─ server/
│  ├─ package.json  tsconfig.json  vitest.config.ts  drizzle.config.ts
│  ├─ drizzle/                   ← SQL migrations
│  ├─ src/
│  │  ├─ index.ts                ← start server + scheduler
│  │  ├─ app.ts                  ← build Fastify app (used by tests too)
│  │  ├─ config.ts               ← env parsing with zod
│  │  ├─ context.ts              ← { db, gateway, clock, bus, log }
│  │  ├─ db/ schema.ts index.ts seed.ts
│  │  ├─ gateway/
│  │  │  ├─ driver.ts            ← GatewayDriver interface
│  │  │  ├─ nftables.ts          ← real driver
│  │  │  ├─ mock.ts              ← in-memory driver
│  │  │  ├─ ruleset.ts           ← builds the base nft script
│  │  │  └─ neighbors.ts         ← IP → MAC lookup (ip neigh / dnsmasq leases)
│  │  ├─ services/
│  │  │  auth.ts customers.ts wallet.ts plans.ts passes.ts sessions.ts devices.ts
│  │  │  promotions.ts giveaways.ts referrals.ts vouchers.ts
│  │  │  momo.ts paystack.ts sms.ts payments.ts
│  │  │  support.ts faq.ts announcements.ts
│  │  │  settings.ts audit.ts reports.ts staff.ts
│  │  ├─ routes/
│  │  │  captive.ts portal.ts admin.ts momo.ts paystack.ts support.ts public.ts events.ts
│  │  ├─ jobs/ scheduler.ts tick.ts expiry.ts reconcile.ts paystack-sweep.ts cleanup.ts
│  │  ├─ realtime/ bus.ts sse.ts
│  │  └─ lib/ errors.ts money.ts phone.ts ids.ts time.ts validate.ts
│  └─ test/ *.test.ts
├─ web/
│  ├─ package.json  vite.config.ts  tsconfig.json
│  ├─ portal.html  admin.html
│  └─ src/
│     ├─ shared/  api.ts  money.ts  sse.ts  theme.css  components/
│     ├─ portal/  main.tsx  App.tsx  pages/*.tsx
│     └─ admin/   main.tsx  App.tsx  pages/*.tsx
└─ infra/
   ├─ gateway/ setup.sh  uninstall.sh  status.sh
   │           templates/ dnsmasq.conf.tpl  hostapd.conf.tpl  networkd-lan.network.tpl
   └─ backup/ backup-loop.sh
```

---

## 5. Core concepts

| Concept | Meaning |
|---|---|
| **Customer** | A person with an account (phone number is unique). |
| **Wallet** | Money balance in pesewas. It goes up with top-ups and down when buying plans. It's a ledger: `wallet_tx` rows; `customers.wallet_balance` is a cached sum updated in the same transaction. |
| **Plan** | Something for sale: price, plus time, validity and/or data, speed tier and maximum devices. |
| **Pass** | What a customer owns after buying a plan, redeeming a voucher or receiving a gift. It has remaining seconds and/or remaining bytes and/or an end time. |
| **Device** | A phone or laptop, identified by MAC address (and last IP). It belongs to a customer after they connect it. |
| **Session** | One device using one pass for internet, from Connect until pause, stop or expiry. Each session has a gateway allow entry. |
| **Gateway** | The driver that actually turns internet on and off for an IP+MAC and counts bytes. |

### Pass kinds

| Kind | Clock | Example |
|---|---|---|
| `time` | Seconds remaining count down **only while connected**. Can be paused. Optional `expires_at` ("use within 30 days"). | "5 hours, use within 30 days" |
| `validity` | Runs from first Connect until `ends_at`, whether connected or not. Unlimited data unless `data_bytes` is set. | "24-hour pass", "7-day unlimited" |
| `data` | Bytes remaining count down with use. `expires_at` from purchase or first use. | "2 GB valid 7 days" |

A plan can combine limits, e.g. `validity` 7 days + `data_bytes` 5 GB: the pass ends at whichever limit comes first.

---

## 6. Gateway

### 6.1 Driver interface (`server/src/gateway/driver.ts`)

```ts
export interface ClientKey { ip: string; mac: string }       // ip: IPv4 dotted, mac: lowercase aa:bb:cc:dd:ee:ff
export interface Counters { upBytes: bigint; downBytes: bigint }

export interface GatewayDriver {
  /** Create or replace the whole wifihub table (idempotent). Called at startup and by "Repair network". */
  init(cfg: GatewayConfig): Promise<void>;
  /** Let this device through for `seconds` (kernel timeout), add byte counters, set its speed tier. */
  allow(key: ClientKey, seconds: number, opts: { sessionId: string; tier?: SpeedTier | null }): Promise<void>;
  /** Change the timeout of an existing allow entry (atomic delete + add). */
  extend(key: ClientKey, seconds: number): Promise<void>;
  /** Remove access now, drop counters, and kill open connections (conntrack). */
  revoke(key: ClientKey, sessionId: string): Promise<void>;
  /** Byte counters for all sessions, keyed by sessionId. */
  readCounters(): Promise<Map<string, Counters>>;
  /** Entries currently in the allowed set (to reconcile with the database). */
  listAllowed(): Promise<Array<ClientKey & { secondsLeft: number }>>;
  /** Always-allowed devices (owner's phone, printer, the MoMo relay phone). */
  setBypass(macs: string[]): Promise<void>;
  /** IP → MAC from the neighbour table / DHCP leases. */
  lookupMac(ip: string): Promise<string | null>;
  /** Health: table present, forwarding on, interfaces up. */
  status(): Promise<GatewayStatus>;
}
```

- `GATEWAY_DRIVER=mock`: everything is kept in memory. `lookupMac` returns a stable fake MAC per IP (or reads `X-Mock-Mac` header in dev). Counters can be pushed by a dev-only endpoint `POST /api/dev/traffic`. Used in tests and on Windows or Mac.
- `GATEWAY_DRIVER=nftables`: the real driver below.
- Leave room for future drivers (`mikrotik`, `opnsense`) behind the same interface. Do not build them now.

### 6.2 nftables ruleset (built by `ruleset.ts`, loaded with `nft -f -`)

Values in `{…}` come from config. The app deletes and recreates the table at startup, then **re-adds every active session from the database** (reconcile), so a restart doesn't drop paying customers for longer than a second.

```nft
table inet wifihub {
  # paid devices: IP + MAC must both match; each element expires on its own
  set allowed {
    type ipv4_addr . ether_addr
    flags timeout
  }
  # owner's devices, relay phone, printers: always allowed
  set bypass_mac { type ether_addr; }
  # destinations reachable before login (portal deps, Paystack checkout), filled by dnsmasq nftset= and by the app
  set walled_garden { type ipv4_addr; flags interval, timeout; }

  chain prerouting_nat {
    type nat hook prerouting priority dstnat; policy accept;
    iifname "{LAN_IF}" ip saddr {LAN_SUBNET} jump captive
  }
  chain captive {
    ether saddr @bypass_mac return
    ip saddr . ether saddr @allowed return
    # every DNS query from a not-yet-allowed device goes to our dnsmasq
    udp dport 53 redirect to :53
    tcp dport 53 redirect to :53
    # every plain-HTTP request goes to the portal (this triggers the phone's "Sign in to network" popup)
    ip daddr != @walled_garden tcp dport 80 dnat ip to {PORTAL_IP}:{PORTAL_PORT}
  }

  chain forward {
    type filter hook forward priority filter; policy drop;
    ct state established,related jump acct
    ct state established,related accept
    iifname "{LAN_IF}" ip saddr {LAN_SUBNET} jump from_customers
  }
  chain from_customers {
    # never let customers reach the owner's private network (Mode A: Huawei subnet)
    ip daddr {PRIVATE_BLOCK} drop
    ether saddr @bypass_mac accept
    ip daddr @walled_garden accept
    ip saddr . ether saddr @allowed jump acct
    ip saddr . ether saddr @allowed accept
    # not allowed: HTTPS gets a fast reject instead of a long timeout
    meta l4proto tcp reject with tcp reset
    reject
  }
  # one pair of counter rules per session: "ip saddr X counter comment up:<id>" / "ip daddr X counter comment down:<id>"
  chain acct { }

  chain input_guard {
    type filter hook input priority filter; policy accept;
    # customers may only talk to the server on DHCP, DNS and the portal
    iifname "{LAN_IF}" udp dport { 53, 67 } accept
    iifname "{LAN_IF}" tcp dport { 53, {PORTAL_PORT} } accept
    iifname "{LAN_IF}" ip saddr {LAN_SUBNET} ct state established,related accept
    iifname "{LAN_IF}" ip saddr {LAN_SUBNET} drop
  }

  chain postrouting_nat {
    type nat hook postrouting priority srcnat; policy accept;
    ip saddr {LAN_SUBNET} oifname "{WAN_IF}" masquerade
  }
}
```

Notes for implementation:

- **Mode B:** `LAN_IF == WAN_IF`, `{PRIVATE_BLOCK}` is empty (skip that rule), and `input_guard` only matches `ip saddr` inside the DHCP range (not the owner's static devices). Masquerade still applies so return traffic comes back through the server.
- **The portal listens on `PORTAL_PORT` (default 8080).** The DNAT sends port 80 there. Also add `ip daddr {PORTAL_IP} tcp dport 80 redirect to :{PORTAL_PORT}` so `http://10.50.0.1` works for allowed devices too.
- **Download-direction accounting:** download packets match `ct state established,related` → `acct`, where the `ip daddr X counter` rule counts them.
- **Counters:** `nft -j list chain inet wifihub acct` returns JSON with each rule's `comment` and `counter.bytes`. Parse comments `up:<sessionId>` / `down:<sessionId>`. Deleting uses the rule `handle` from the same listing.
- **`revoke`** runs in one `nft -f -` transaction (delete element + delete both counter rules), then `conntrack -D -s <ip>` and `conntrack -D -d <ip>` so open connections (downloads, video) die at once. Exit code 1 from conntrack ("0 flow entries") is not an error.
- **`extend`** runs `delete element … ; add element … { ip . mac timeout Ns }` in one `nft -f -` transaction.
- **IPv6:** the customer network is IPv4 only. dnsmasq hands out no IPv6 and the table drops IPv6 forwarding from LAN_IF (`meta nfproto ipv6 iifname LAN_IF drop` in `forward`). This prevents IPv6 bypass.
- **All values are validated before they're written into a script:** IP with a strict IPv4 regex inside `LAN_SUBNET`, MAC with `^([0-9a-f]{2}:){5}[0-9a-f]{2}$`, seconds as an integer 1–31,536,000, sessionId as a UUID.
- Serialise all `nft` calls through one async queue (mutex) so two transactions never interleave.

### 6.3 Captive portal detection

Phones test a known URL right after joining Wi-Fi. While the device is not allowed, every port-80 request lands on the app, which answers **any Host** like this:

| Request | Response |
|---|---|
| Host is the portal (`PORTAL_IP`, `wifi.hub`) | serve the portal normally |
| Any other host, any path (e.g. `connectivitycheck.gstatic.com/generate_204`, `captive.apple.com/hotspot-detect.html`, `www.msftconnecttest.com/connecttest.txt`) | `302 Location: http://{PORTAL_HOST}/portal/?from=<url-encoded original>` with `Cache-Control: no-store` |

Once the device is allowed, its traffic no longer reaches the app, the real check succeeds and the phone closes the popup. The portal's success page also links to "Continue to the internet".

Implement this as the **first** route hook in Fastify: if `Host` is not a portal host and the path is not `/api/*`, redirect.

**RFC 8910 (DHCP option 114) captive-portal API:** dnsmasq sends `dhcp-option=114,"http://{PORTAL_HOST}/api/captive"`. `GET /api/captive` returns `application/captive+json`:
`{"captive": <bool for this IP>, "user-portal-url": "http://{PORTAL_HOST}/portal/", "seconds-remaining": <n>}`. Newer Android and iOS use this for a cleaner popup.

### 6.4 Identifying the device

On every portal API call, the server knows the client IP (`request.ip`; the app is on host network and DNAT keeps the source). `lookupMac(ip)` reads `ip -j neigh show <ip>`, falling back to the dnsmasq leases file (`/var/lib/misc/dnsmasq.leases`, mounted read-only). Requests from outside `LAN_SUBNET` (e.g. through Cloudflare) are "remote": the customer can see their account and top up but can't press Connect.

**MAC randomisation:** modern phones use a random MAC per Wi-Fi network. Usually it's stable for that network; iOS 18 "Rotating" and some Android versions change it. Rules:

- Devices are matched by MAC. A new MAC is a new device row, which the customer connects again after logging in.
- `max_devices` on a pass counts **active sessions**, not device rows, so a rotated MAC doesn't use up a slot.
- The portal shows a short tip: "If you keep getting logged out, turn off *Private Wi-Fi address* for ZENAA WiFi."

### 6.5 Walled garden (reachable before login)

- Always: the portal itself, plus DNS and DHCP on the server.
- Paystack checkout: `checkout.paystack.com`, `js.paystack.co`, `api.paystack.co`, `standard.paystack.co`, `paystack.com`, plus any extra domains the owner adds in Settings (card 3-D Secure pages vary by bank).
- Implementation: dnsmasq resolves these domains and adds their IPs to the nft set automatically (`nftset=/checkout.paystack.com/js.paystack.co/.../4#inet#wifihub#walled_garden`, needs dnsmasq ≥ 2.87). The app regenerates `/etc/dnsmasq.d/wifihub-garden.conf` from Settings (mounted rw). dnsmasq does not re-read its config files on `SIGHUP`, so the app then asks a tiny host helper to run `systemctl restart dnsmasq` (`infra/gateway/wifihub-helper`, see §19.3). If the helper is absent, show "Restart dnsmasq to apply" in the admin.
- Default: walled-garden **off for Paystack** until the owner enables online payment before login. MoMo direct needs no walled garden, because the customer pays on their own phone network, not over Wi-Fi.

### 6.6 Speed tiers (Phase 6)

Each plan may have a `speed_tier` (e.g. Basic 2 Mbps down / 1 Mbps up, Premium 10/5). Implement with nftables per-IP rate limiting using dynamic sets:

```nft
set tier_<id> { type ipv4_addr; flags timeout; }          # members = IPs on this tier (same timeout as allowed)
set tier_<id>_dn { type ipv4_addr; size 1024; flags dynamic, timeout; timeout 10s; }
# in from_customers before accept / in forward for established download:
ip daddr @tier_<id> update @tier_<id>_dn { ip daddr limit rate over {DN_KBYTES} kbytes/second burst {BURST} kbytes } drop
```

Verify the exact syntax against the installed `nft --version` (≥ 1.0) and write a test script `infra/gateway/test-speed.sh`. If policing performs badly, switch to `tc` HTB with an `ifb` device. Record the decision in DECISIONS.md. This is **optional**: everything else must work without tiers.

### 6.7 Fail-safe behaviour

| Situation | Result |
|---|---|
| App crashes | Allowed devices keep working until their kernel timeout (= remaining time). New logins impossible. Data passes have a timeout of at most 15 minutes and are re-extended by the tick, so data can't overrun for long. |
| Server reboots | Table is gone, so everyone is blocked. dnsmasq still runs. On app start: `init()` + reconcile re-allows active sessions. |
| Postgres down | Tick skips; gateway entries expire on their own. |
| Clock jump | Time is deducted from the monotonic difference between ticks, capped at 2 × tick interval per tick. |

---

## 7. Database

All tables have `id uuid pk default gen_random_uuid()` and `created_at timestamptz default now()` unless noted. Money is `integer` (pesewas), bytes are `bigint`, seconds are `integer`. Use drizzle-kit migrations. A `deleted_at` column means a soft delete.

### Accounts

- **customers**: `full_name`, `phone` (unique, E.164 e.g. `+233241234567`), `email` (nullable), `password_hash`, `wallet_balance` (int, cached), `status` (`active` | `blocked`), `phone_verified_at`, `referral_code` (unique, 6 chars, no 0/O/1/I), `referred_by` (fk customers), `notes` (staff), `last_seen_at`, `deleted_at`.
- **customer_sessions_auth**: `customer_id`, `token_hash`, `user_agent`, `ip`, `expires_at`, `revoked_at` (login tokens; lets staff "sign out everywhere").
- **otp_codes**: `phone`, `purpose` (`verify` | `reset`), `code_hash`, `expires_at`, `attempts`, `used_at`.
- **staff**: `username` (unique), `full_name`, `password_hash`, `role` (`owner` | `admin` | `cashier` | `support`), `pin_hash` (optional), `active`.

### Products

- **plans**: `name`, `description`, `kind` (`time` | `validity` | `data`), `price` (int), `time_seconds` (nullable), `validity_seconds` (nullable), `data_bytes` (nullable), `use_within_days` (nullable; time/data passes expire this many days after purchase), `max_devices` (default 1), `speed_tier_id` (nullable), `visible` (bool), `sort`, `badge` (e.g. "Best value"), `archived_at`.
- **speed_tiers**: `name`, `down_kbps`, `up_kbps`.

### Passes and usage

- **passes**: `customer_id`, `plan_id` (nullable for gifts), `source` (`purchase` | `voucher` | `trial` | `giveaway` | `referral` | `staff_gift` | `promo`), `source_id`, `kind`, `seconds_left` (nullable), `bytes_left` (nullable), `ends_at` (nullable; validity end or use-within expiry), `first_used_at`, `max_devices`, `speed_tier_id`, `status` (`active` | `used_up` | `expired` | `revoked`), `label` (shown to customer, e.g. "1 Hour · from voucher").
- **pass_tx**: ledger: `pass_id`, `delta_seconds`, `delta_bytes`, `reason` (`grant` | `usage` | `adjust` | `refund` | `expire`), `session_id`, `actor_staff_id`, `note`.
- **devices**: `customer_id` (nullable until claimed), `mac` (unique), `last_ip`, `label` (customer-set, e.g. "My Tecno"), `vendor` (optional), `bypass` (bool; owner devices), `blocked` (bool), `last_seen_at`.
- **sessions**: `customer_id`, `device_id`, `pass_id`, `ip`, `mac`, `status` (`active` | `paused` | `ended`), `started_at`, `last_tick_at`, `last_traffic_at`, `ended_at`, `end_reason` (`time_up` | `data_up` | `expired` | `customer_stop` | `idle` | `staff_kick` | `blocked` | `replaced` | `gateway_lost`), `seconds_used`, `bytes_up`, `bytes_down`, `counter_up_last` and `counter_down_last` (bigint; last raw counter reading for delta maths).

### Money

- **wallet_tx**: `customer_id`, `amount` (+ credit, − debit), `balance_after`, `kind` (`topup_momo` | `topup_paystack` | `topup_cash` | `topup_voucher` | `bonus` | `purchase` | `refund` | `adjust`), `method_ref` (txn ID / Paystack reference), `payment_id` (nullable), `purchase_id` (nullable), `actor_staff_id`, `note`. Unique index on (`kind`, `method_ref`) where `method_ref` is not null, so nothing is credited twice.
- **payments**: one row per money-in event for reports: `customer_id`, `method` (`momo_direct` | `paystack` | `cash` | `voucher`), `amount`, `fee` (Paystack fee if known), `reference`, `status`, `staff_id`, `shift_id`.
- **purchases**: `customer_id`, `plan_id`, `list_price`, `discount_total`, `paid` (= list − discount), `promo_code_id`, `promotion_ids` (uuid[]), `pass_id`, `refunded_at`.
- **momo_requests**: (as in ZENAA) `customer_id`, `amount`, `network`, `code` (6 digits, unique while pending), `status` (`pending` | `confirmed` | `rejected` | `expired` | `cancelled`), `txn_id` (unique when not null), `sms_id`, `confirmed_via` (`sms` | `staff`), `confirmed_by`, `confirmed_at`, `expires_at`, `note`, `payment_id`.
- **momo_sms**: `sender`, `body`, `amount`, `txn_id` (unique when not null and status != ignored), `reference`, `payer_name`, `payer_phone`, `status` (`unmatched` | `matched` | `ignored` | `credited_manually`), `device`, `received_at`, `request_id`.
- **relay_devices**: `device` (name), `token_hash`, `last_seen_at`, `revoked_at`.
- **sms_outbox**: `phone`, `body`, `kind` (`momo` | `topup` | `otp` | `low_balance` | `expired` | `test` | `broadcast`), `customer_id`, `status` (`queued` | `sending` | `sent` | `failed` | `expired`), `attempts`, `claimed_at`, `sent_at`, `device`, `error`.
- **paystack_tx**: `customer_id`, `reference` (unique, `wh_` + random), `amount`, `currency`, `status` (`pending` | `success` | `failed` | `abandoned` | `expired`), `channel`, `paystack_id`, `fees`, `verified_at`, `raw` (jsonb, last verify response), `credited_wallet_tx_id`.
- **shifts**: `staff_id`, `opened_at`, `closed_at`, `opening_cash`, `counted_cash`, `expected_cash`, `momo_counted`, `note`.

### Promotions and giveaways

- **promo_codes**: `code` (unique, uppercase), `description`, `effect` (`percent_off` | `amount_off` | `bonus_seconds` | `bonus_bytes` | `free_plan` | `wallet_credit`), `value` (int: percent ×1, pesewas, seconds or bytes), `plan_id` (for `free_plan`), `applies_to_plan_ids` (uuid[], empty = all), `min_spend`, `starts_at`, `ends_at`, `max_uses`, `max_uses_per_customer` (default 1), `first_purchase_only`, `new_customers_days` (nullable: only accounts younger than N days), `active`, `uses_count`.
- **promo_redemptions**: `promo_code_id`, `customer_id`, `purchase_id` (nullable), `value_given`. Unique (`promo_code_id`, `customer_id`, `purchase_id`).
- **promotions** (automatic, no code): `name`, `type` (`happy_hour` | `topup_bonus` | `first_topup_bonus` | `plan_sale`), `percent` (int), `bonus_amount` (int), `min_amount`, `applies_to_plan_ids`, `days_of_week` (int[] 0–6), `time_from`, `time_to` (HH:MM local), `starts_at`, `ends_at`, `active`, `banner_text`.
- **giveaways**: `title`, `description`, `reward_kind` (`pass` | `wallet_credit`), `plan_id` / `seconds` / `bytes` / `amount`, `total_slots`, `claimed_count`, `per_customer` (default 1), `starts_at`, `ends_at`, `requires_verified_phone`, `active`.
- **giveaway_claims**: `giveaway_id`, `customer_id`, `device_mac`, `pass_id`. Unique (`giveaway_id`, `customer_id`).
- **trial_claims**: `customer_id`, `device_mac`, `day` (date, local). Unique (`device_mac`, `day`) and unique (`customer_id`, `day`).
- **referrals**: `referrer_id`, `referee_id` (unique), `status` (`pending` | `rewarded` | `void`), `rewarded_at`.
- **voucher_batches**: `name`, `plan_id` or `wallet_amount`, `count`, `price_each` (what the owner sells them for; 0 = giveaway), `expires_at`, `created_by`, `note`.
- **vouchers**: `batch_id`, `code` (unique, format `XXXX-XXXX` from alphabet `ABCDEFGHJKLMNPQRSTUVWXYZ23456789`), `status` (`unused` | `redeemed` | `void`), `redeemed_by`, `redeemed_at`, `printed` (bool).

### Support and content

- **tickets**: `customer_id`, `subject`, `category` (`payment` | `connection` | `account` | `other`), `status` (`open` | `waiting_customer` | `closed`), `priority` (`normal` | `high`), `assigned_staff_id`, `related_momo_request_id`, `related_paystack_tx_id`, `last_message_at`, `unread_for_staff` (bool), `unread_for_customer` (bool).
- **ticket_messages**: `ticket_id`, `from` (`customer` | `staff` | `system`), `staff_id`, `body` (≤ 2000), `created_at`.
- **faq**: `question`, `answer`, `sort`, `visible`.
- **announcements**: `text`, `level` (`info` | `promo` | `warning`), `starts_at`, `ends_at`, `active`.

### System

- **settings**: single row `key='app'`, `value jsonb` validated by a zod schema (see §18.2).
- **audit_log**: `actor_type` (`staff` | `customer` | `system` | `relay`), `actor_id`, `action` (e.g. `wallet.adjust`), `target_type`, `target_id`, `data` jsonb, `ip`, `created_at`.

---

## 8. Business rules: money, time and data

### 8.1 Buying a plan (`POST /api/portal/purchases`)

1. Load the plan (visible, not archived) and the customer (active).
2. Compute the price (§10.3): list price − best automatic promotion − promo code, never below 0.
3. In one transaction:
   - lock the customer row (`SELECT … FOR UPDATE`);
   - check `wallet_balance ≥ paid`, otherwise return `402` "Not enough money. Top up GHS X.XX more" with the shortfall amount (the portal then offers to top up that amount);
   - add a `wallet_tx` (−paid, `purchase`) and a `purchases` row;
   - create the `passes` row (and a `pass_tx` `grant`):
     - `time`: `seconds_left = plan.time_seconds (+ bonus seconds)`, `ends_at = now + use_within_days` if set;
     - `validity`: `seconds_left = null`, `ends_at = null` until first use (set at first Connect to `first_used_at + validity_seconds`);
     - `data`: `bytes_left = plan.data_bytes (+ bonus)`, `ends_at = now + use_within_days`;
     - copy `max_devices` and `speed_tier_id`;
   - record promo redemption and promotion IDs; write audit.
4. If the request says `connectNow: true` and comes from a device on the Wi-Fi, start a session (§8.2).

### 8.2 Connect (`POST /api/portal/connect`)

1. The request must come from `LAN_SUBNET`. Look up the MAC; if not found, return `409` "Can't identify your device. Turn Wi-Fi off and on, then try again."
2. Upsert `devices` (claim it for the customer; if another customer owns it, re-assign and end that customer's session on it with reason `replaced`).
3. Pick the pass: the one given in the request, else the **best active pass**: validity passes first (already paid by the day), then the time or data pass that expires soonest, then the oldest.
4. Check `max_devices`: active sessions on that pass < `max_devices`, otherwise `409` "This pass is already used on N device(s). Disconnect one first", listing them with a Disconnect button.
5. Compute the **kernel timeout**:
   - `time`: `seconds_left`, also capped by `ends_at` if set;
   - `validity`: `ends_at − now` (setting `ends_at` on first use);
   - `data`: `min(15 min, ends_at − now)`, re-extended by the tick while bytes remain.
6. `gateway.allow({ip, mac}, timeout, {sessionId, tier})`, then insert the session (`active`). If the gateway call fails, roll back and return `503` "Network is not ready, please tell staff".
7. Return the session, and the portal shows the success screen with a live countdown.

If the device already has an active session, Connect returns it (idempotent). If it has a **paused** session on a usable pass, Connect resumes it.

### 8.3 The tick (every 15 s, `jobs/tick.ts`)

For each `active` session:

1. `elapsed = min(now − last_tick_at, 2 × tick)`.
2. Read counters (one `readCounters()` call per tick for all sessions). Compute `Δup`, `Δdown` from the last raw values; a counter lower than the last value means it was recreated, so treat the delta as the new value.
3. If `Δup + Δdown > IDLE_BYTES_THRESHOLD` (default 20 KB per tick), set `last_traffic_at = now`.
4. Deduct in one transaction:
   - `time` pass: `seconds_left −= elapsed`;
   - `data` pass: `bytes_left −= (Δup + Δdown)`;
   - `validity` pass with data cap: `bytes_left −= …`;
   - write a `pass_tx` `usage` row **aggregated per minute** (update the current minute's row instead of inserting every tick), and update the session totals.
5. Decide:
   - `seconds_left ≤ 0`, `bytes_left ≤ 0` or `now ≥ ends_at` → mark the pass `used_up`/`expired`. If `settings.autoContinue` is on and the customer has another usable pass, switch the session to it (`extend` the gateway entry); otherwise `revoke` and end the session with `time_up` / `data_up` / `expired`.
   - **Idle** (time passes only): `now − last_traffic_at ≥ idleMinutes` (default 5): `revoke`, set session `paused`, `end_reason='idle'`, and **refund** the idle seconds after the last traffic (add back `now − last_traffic_at − 60 s grace`, never more than what was deducted). Validity and data passes are never paused for idleness (their clock doesn't depend on it).
   - Data passes: when the gateway timeout has < 5 min left and bytes remain, call `extend`.
6. Warnings (once per session per threshold): at 10 min and 2 min left, or at 100 MB and 20 MB left, push an SSE event to the portal; if `settings.smsLowBalance`, queue one SMS at the first threshold.

**Overshoot:** data can overrun by at most one tick of traffic (≈15 s). Clamp `bytes_left` at 0, don't make it negative, and don't charge the overrun.

### 8.4 Pause, stop and resume

- **Pause** (customer, time passes only): `revoke`, session `paused`, keep the device link. Next Connect resumes it.
- **Disconnect** (customer, any pass): `revoke`, session `ended` (`customer_stop`). A validity pass keeps running anyway, and the portal says so before confirming.
- **Kick** (staff): `revoke`, `ended` (`staff_kick`); optionally block the device or customer.
- Blocking a customer ends all their sessions (`blocked`) and signs them out.

### 8.5 Expiry job (every minute)

- Passes past `ends_at` → `expired`, ending sessions.
- MoMo requests older than 2 h → `expired` (still matchable by a late SMS for 48 h, then rejected).
- Paystack transactions pending > 24 h → `expired`.
- OTPs and auth tokens past expiry are deleted.

### 8.6 Reconcile (at startup and every 5 min)

- For each DB `active` session: make sure the gateway has it, re-adding it with the correct remaining timeout if missing.
- For each gateway entry with no active DB session: revoke it.
- A session whose device IP changed (new DHCP lease): revoke the old IP and allow the new one when the device next hits the portal.

### 8.7 Refunds and adjustments

- Staff can refund a purchase only if the pass is unused (`first_used_at` null and `seconds_left` and `bytes_left` unchanged), returning money to the wallet. Otherwise use a manual wallet adjustment with a reason (needs `owner` or `admin`).
- Staff can **gift** time or data (creates a pass with source `staff_gift`) and adjust a pass (± seconds or bytes) with a reason. Everything is audited.
- Wallet money is **not withdrawable as cash** by default. Setting `walletCashOut` (default off) allows staff to pay it out.

---

## 9. Payments

All payment methods put **money into the wallet first**. Plans are bought from the wallet. This keeps one simple flow and lets promotions work the same way for every method.

`services/payments.ts` defines:

```ts
interface TopupMethod {
  id: 'momo_direct' | 'paystack' | 'voucher' | 'cash';
  enabled(settings): boolean;
}
creditWallet(tx, { customerId, amount, kind, methodRef, paymentMethod, actor, note }) // the only way money enters
```

`creditWallet` also applies **top-up promotions** (§10.2): it adds a separate `bonus` wallet_tx. It queues the thank-you SMS and emits `wallet.changed` on the bus.

### 9.1 MoMo direct (no gateway, no fee), same design as ZENAA GameHub

**Flow:**

1. Portal **Top up → MoMo**: the customer picks an amount (quick buttons GHS 2, 5, 10, 20, 50, or custom within `momoMinTopup`–`momoMaxTopup`) and a network.
2. The server creates a `momo_request` with a **6-digit numeric code** (100000–999999, unique among pending requests, at most 3 pending per customer, valid 2 h) and shows:
   - "Send **GHS 10.00** to **024 000 0000 (Adamu …, MTN MoMo)**"
   - "Reference: **482193**"
   - step-by-step for the network: MTN `*170#` → 1 Transfer Money → 1 MoMo User → number → amount → reference; Telecel `*110#`; AT `*110#`. Steps text lives in settings so the owner can edit it.
   - A **"Copy number"** button, a **"Dial *170#"** link (`tel:*170%23`), and a waiting animation that listens on SSE.
3. The owner's **MoMo relay phone** (Android) receives the network's "Payment received…" SMS and forwards it to `POST /api/momo/sms`.
4. The server parses it (§9.1.2), checks the sender is trusted, and matches it **by code**, else **by transaction ID**, else **by payer phone = customer phone + exact amount + exactly one pending request**.
5. **Auto-credit rules** (settings): credit at once if `momoAutoConfirm` is on, amount < `momoAutoMax` (default GHS 300), and the time is inside `momoAutoFrom`–`momoAutoTo` (if `momoAutoHoursOn`). Otherwise the request becomes "matched, waiting for staff".
6. The portal turns green ("GHS 10.00 received ✓"), the wallet updates, and the thank-you SMS is queued to the relay phone's outbox.
7. Anything unmatched waits in **Admin → MoMo inbox → Unclaimed SMS**. Staff can credit it to a customer or ignore it. Waiting requests can be confirmed by staff after checking the phone (requires permission `momo.confirm`). Customers can also type the **transaction ID** from their own SMS to claim a payment (`POST /api/portal/momo/:id/txn`), which matches it against unclaimed SMS.

**Relay phone API**, compatible with the **ZENAA MoMo** Android app, so the same APK works:

| Method & path | Auth | Purpose |
|---|---|---|
| `POST /api/admin/momo/relay/code` | staff `settings.manage` | create a 6-digit pairing code (valid 10 min) |
| `POST /api/momo/relay/pair` `{code, device}` | none (8 wrong tries per IP → 15 min lock) | returns `{token, centerName}`; store sha256(token) |
| `POST /api/admin/momo/relay/link` `{device}` | staff | token + URL for third-party "SMS forwarder" apps: `/api/momo/sms?token=…` |
| `POST /api/momo/sms` | `Authorization: Bearer <token>` or `?token=` | body accepts `{from|sender|address|phone, body|text|message|msg|content}` |
| `GET /api/momo/relay/ping` | token | `{ok, device, centerName}` |
| `GET /api/momo/relay/outbox` | token | up to 5 queued SMS `{messages:[{id,to,body}]}` and marks them `sending` |
| `POST /api/momo/relay/outbox/:id` `{ok, error?}` | token | delivery report; retry up to 3 times, then `failed` |
| `DELETE /api/admin/momo/relay/:id` | staff | revoke a phone |

The **relay phone must be in `bypass_mac`** (always allowed), or connected to the owner's own network.

The same outbox sends **OTP codes** (sign-up verification, password reset), **low-balance** and **time's up** messages, all from the relay phone's SIM. This means no paid SMS provider is needed.

#### 9.1.1 Safety rules (all required)

| Risk | Rule |
|---|---|
| Fake "Payment received" SMS sent from a normal phone | Only senders in `momoSmsSenders` count (alphanumeric names like `MobileMoney`, `T-CASH`). Senders that look like phone numbers are **always** rejected. |
| Same SMS forwarded twice | Unique `txn_id`. A duplicate is stored as `ignored`. |
| Wrong amount | Must match exactly. Otherwise the request waits with the note "SMS amount differs (GHS 9.00 vs 10.00)". |
| Large amounts / night time | Auto-credit limits (above). |
| Staff crediting without money | Permission `momo.confirm`, optional staff PIN, audit row with staff name; shift close compares the MoMo total. |

#### 9.1.2 SMS parser: port this exactly (from ZENAA GameHub `server/src/services/momo.ts`)

```ts
export interface ParsedSms {
  incoming: boolean;
  amountMajor: number | null;   // major units as written ("1,250.50" → 1250.5)
  txnId: string | null;
  reference: string | null;
  payerName: string | null;
  payerPhone: string | null;
}

const CUR = '(?:GHS|GH₵|GH¢|GHC|GH\\s?C|₵|¢|NGN|₦|KES|KSh|UGX|TZS|RWF|XOF|XAF|ZMW|MWK|USD|\\$)';

/** Reads a "money received" SMS from MTN MoMo, Telecel Cash, AT Money and similar networks. */
export function parseMomoSms(text: string): ParsedSms {
  const t = text.replace(/\s+/g, ' ').trim();
  const incoming =
    /(payment (has been )?received|you have received|received (a )?(payment|money)|cash ?in received|has been credited|credited to your|deposit received|received\s+(GHS|GH₵|GH¢|GHC|₵|¢))/i.test(t) &&
    !/(you have sent|payment made for|you have paid|withdrawal of|cash ?out of|transferred to)/i.test(t);

  // Amount: the first currency amount after "received", else the first one at all (balances come later).
  const re = new RegExp(`${CUR}\\s?([0-9][0-9,]*(?:\\.[0-9]{1,2})?)`, 'ig');
  const all = [...t.matchAll(re)].map((m) => ({ at: m.index ?? 0, v: Number(m[1].replace(/,/g, '')) }));
  const rx = /received/i.exec(t);
  const after = rx ? all.find((a) => a.at > rx.index) : undefined;
  const amountMajor = (after ?? all[0])?.v ?? null;

  const txn =
    /(?:financial transaction id|transaction id|trans(?:action)?\.?\s?id|txn\s?id|transaction no\.?|trx\s?id|ref(?:erence)? no\.?)\s*[:#-]?\s*([A-Z0-9][A-Z0-9.]{5,30})/i.exec(t) ??
    /^([0-9]{8,})\s+confirmed/i.exec(t);
  const ref = /\breference\s*[:-]?\s*([^.;\n]{1,40}?)(?=\s*(?:[.;]|$|transaction|current|your|fee|available|balance|tran))/i.exec(t);
  const phone = /(?:from|by)\s+(?:[A-Za-z .'-]{2,40}?\s+)?\(?(\+?233\d{9}|0\d{9}|\+?\d{10,13})\)?/i.exec(t);
  const name =
    /from\s+(?:\+?\d{9,13}\s*[-–]\s*)?([A-Z][A-Za-z .'-]{1,40}?)(?=\s*(?:\(|\d|on\s|\.|,|current|reference|$))/i.exec(t);
  return {
    incoming,
    amountMajor: amountMajor != null && Number.isFinite(amountMajor) ? amountMajor : null,
    txnId: txn ? txn[1].replace(/\.$/, '').toUpperCase() : null,
    reference: ref ? ref[1].trim() : null,
    payerName: name ? name[1].trim() : null,
    payerPhone: phone ? phone[1] : null,
  };
}

const norm = (s: string) => s.toLowerCase().replace(/[^a-z0-9]/g, '');
export function trustedSender(sender: string, settings: { momoSmsSenders: string[] }) {
  const s = sender.trim();
  if (/^\+?[\d\s-]{6,}$/.test(s)) return false; // a phone number: anyone can send those
  return settings.momoSmsSenders.some((x) => norm(x) === norm(s));
}

const NUM_CODE_RE = /(?<!\d)([1-9]\d{5})(?!\d)/g;
/** Codes are only read from the Reference part; spaces, dots or dashes typed inside the code are ignored. */
export function codesInSms(reference: string | null) {
  const squeezed = (reference ?? '').replace(/(\d)[\s.-]+(?=\d)/g, '$1');
  return [...new Set([...squeezed.matchAll(NUM_CODE_RE)].map((m) => m[1]))];
}
```

Tests must cover real-looking samples:

- MTN: `Payment received for GHS 10.00 from KWAME MENSAH 0241234567 Current Balance: GHS 152.30. Available Balance: GHS 152.30. Reference: 482193. Transaction ID: 51234567890. Fee charged: GHS0.00.`
- Telecel: `You have received GHS 5.00 from 0201234567 - AMA SERWAA on 2026-11-25 at 10:15. Reference: 4821 93. Transaction ID: 0001234567.`
- An outgoing `You have sent GHS 5.00 to …` → `incoming=false`.
- A trusted sender that is a phone number → rejected.
- A duplicate transaction ID → ignored.

### 9.2 Paystack (card and mobile money, online)

Settings: `paystackEnabled`, `paystackPublicKey`, `paystackSecretKey` (stored encrypted with `APP_SECRET` using AES-256-GCM; shown masked as `sk_live_…abcd`), `publicUrl` (the HTTPS address through Cloudflare, needed for webhooks and the callback).

**Flow:**

1. `POST /api/portal/topups/paystack {amount}`: create `paystack_tx` with `reference = "wh_" + 16 random base32 chars`. Call
   `POST https://api.paystack.co/transaction/initialize`, header `Authorization: Bearer <secret>`, body:
   `{ email, amount (pesewas), currency: "GHS", reference, callback_url: "<portal origin>/portal/paid?ref=<reference>", channels: ["card","mobile_money"], metadata: { customer_id, purpose: "wallet_topup" } }`.
   Email: the customer's email, or else `"<phone digits>@<PAYSTACK_FALLBACK_EMAIL_DOMAIN>"` (owner sets a domain they own).
   Return `authorization_url`; the portal redirects there.
2. **Verify** (one function, `verifyAndCredit(reference)`, which credits exactly once): `GET https://api.paystack.co/transaction/verify/<reference>`. Credit only if `data.status === "success"`, `data.amount === tx.amount`, `data.currency === tx.currency` and `data.reference === reference`. Use a conditional `UPDATE paystack_tx SET status='success' … WHERE id=? AND status='pending' RETURNING`; if no row comes back, the payment is already handled. Then `creditWallet(kind='topup_paystack', methodRef=reference)`. A mismatch sets `failed` with a note and raises an admin alert.
3. `verifyAndCredit` is called from four places:
   - the portal return page `/portal/paid?ref=…`;
   - the portal polling `GET /api/portal/topups/paystack/:ref` every 4 s;
   - the **webhook** `POST /api/paystack/webhook`: verify `x-paystack-signature` = HMAC-SHA512(raw body, secret key) with `timingSafeEqual`, and handle `charge.success` by calling verify (don't trust the webhook body alone). Reply 200 fast;
   - a **sweep job** every minute for pending transactions younger than 24 h.
4. Counter top-up: Admin → Customer → "Paystack QR": the server creates the transaction and shows a QR code of `authorization_url`; the customer pays on their phone and the dialog turns green via SSE.
5. **Before login on the Wi-Fi**, Paystack needs the walled garden (§6.5). Otherwise customers pay on mobile data or use MoMo direct.

Admin Paystack page: list with status, channel, fee, reference; a "Test connection" button that calls `GET /bank?currency=GHS` or `/balance` with the key.

### 9.3 Vouchers (printed cards and QR)

- **Admin → Vouchers → New batch**: plan (or wallet amount), count (1–1000), sale price (0 = giveaway), expiry, note. Codes are `XXXX-XXXX` from `ABCDEFGHJKLMNPQRSTUVWXYZ23456789` (≈ 1.1 × 10¹² combinations), generated with `crypto.randomInt`.
- **Print sheet** (`/admin/vouchers/:batchId/print`): an A4 grid of cards (e.g. 3 × 8), each showing brand, plan name, price, code, expiry, the Wi-Fi name, and a **QR** encoding `http://{PORTAL_HOST}/portal/redeem?code=XXXX-XXXX`. Add a CSV export.
- **Wi-Fi join QR** (poster): `WIFI:T:nopass;S:<SSID>;;` (or `T:WPA;P:<pw>` if the SSID has a password). Admin → Posters prints this QR plus the portal QR.
- **Redeem** (`POST /api/portal/vouchers/redeem {code}`): normalise: uppercase, remove spaces, insert the dash after 4 characters. The alphabet has no 0, O, 1 or I, so there's no look-alike mapping; unknown codes are simply rejected. Rate limit: 10 tries per customer per hour and 20 per IP per hour; lock for 1 h after that. In one transaction: `UPDATE vouchers SET status='redeemed' … WHERE code=? AND status='unused' RETURNING`, then grant the pass (source `voucher`) or credit the wallet (`topup_voucher`). Sold vouchers count as revenue **when sold** (batch `price_each × count` recorded by staff at sale), not when redeemed. The report shows both.
- **Guest redeem without an account** (setting `voucherGuestMode`, default on): on the captive portal, "I have a voucher" → code → the system creates a **guest customer** (`phone` null, name "Guest XXXX") linked to that device, and connects at once. Later the guest can "Save my account" (add phone + password) to keep leftover time.

### 9.4 Cash at the counter

Admin → Customer → **Add money**: amount, method cash, note. Requires an open shift if `requireOpenShift` is on, and permission `wallet.topup`. It creates `wallet_tx` `topup_cash` and a `payments` row with `shift_id`. Shift close shows expected cash vs counted.

---

## 10. Discounts and promotions

### 10.1 Promo codes (customer types a code)

Entered on the plan checkout sheet or the Deals page. Validation (in this order, each with a friendly message): exists and active → inside the date window → `uses_count < max_uses` → customer's uses < `max_uses_per_customer` → `first_purchase_only` (no previous purchases) → `new_customers_days` → plan is in `applies_to_plan_ids` → `price ≥ min_spend`.

Effects:

| Effect | Result |
|---|---|
| `percent_off` | price × (100 − value) / 100, rounded **down** to whole pesewas for the discount (customer-friendly: round the final price down) |
| `amount_off` | price − value, minimum 0 |
| `bonus_seconds` / `bonus_bytes` | full price, the pass gets extra |
| `free_plan` | redeemed on the Deals page without a purchase; grants the plan's pass (source `promo`) |
| `wallet_credit` | redeemed on the Deals page; adds `bonus` money to the wallet |

`uses_count` increments inside the purchase transaction with `UPDATE … WHERE uses_count < max_uses RETURNING`, so the last slot can't be taken twice.

### 10.2 Automatic promotions (no code)

| Type | Example | Applies at |
|---|---|---|
| `happy_hour` | 30% off 1-hour plans, Mon–Fri 06:00–09:00 | purchase price |
| `plan_sale` | Day pass GHS 8 instead of 10 this week | purchase price |
| `topup_bonus` | Top up GHS 50 or more, get 10% extra | `creditWallet` (bonus wallet_tx) |
| `first_topup_bonus` | First top-up gets GHS 2 extra | `creditWallet` |

Time windows use the setting `timezone` (Africa/Accra). Windows that cross midnight work (e.g. 22:00–02:00).

### 10.3 Price calculation (`promotions.ts → quote(customer, plan, code?)`)

1. `list = plan.price`.
2. Best single automatic price promotion (`happy_hour` / `plan_sale`) → `afterAuto`.
3. Promo code (if any). By default **don't stack** with automatic promotions: use whichever gives the lower price. Setting `stackPromotions` = true applies the code on top of `afterAuto`.
4. `paid = max(0, result)`. Return `{list, discounts:[{label, amount}], paid, bonusSeconds, bonusBytes}`.

The portal calls `GET /api/portal/quote?planId=&code=` to show the crossed-out price before buying. The purchase endpoint **recomputes** the quote and never trusts a price from the client.

### 10.4 Deals page (customer)

- Active automatic promotions with countdown ("Happy hour ends in 1h 12m").
- Promo code box.
- Free daily trial button (§11.1).
- Active giveaways with "Claim" and slots left.
- Referral card: "Invite friends: you both get 30 minutes" with share buttons (WhatsApp `https://wa.me/?text=…`, copy link `http://{publicUrl or portal}/portal/signup?ref=CODE`).

---

## 11. Giveaways, free trial and referrals

### 11.1 Free daily trial

Settings: `trialEnabled` (default on), `trialMinutes` (15), `trialRequiresAccount` (true), `trialRequiresVerifiedPhone` (false), `trialSpeedTierId` (optional, slower).

- One claim per **device MAC per local day** and one per **customer per local day** (both unique indexes).
- Creates a `time` pass (source `trial`, `max_devices` 1, `ends_at` = end of today) and connects at once.
- Abuse guard: if one customer's trials appear on more than 3 different MACs in 7 days, stop giving them trials and flag them in the admin.

### 11.2 Giveaway campaigns

Admin creates: "First 50 customers get 1 hour free this Friday". Fields as in §7. Claiming is a transaction with `UPDATE giveaways SET claimed_count = claimed_count + 1 WHERE id=? AND claimed_count < total_slots RETURNING`, and a unique claim per customer. The reward is a pass or wallet credit. The portal shows the remaining slots live (SSE).

### 11.3 Referrals

- Each customer has a `referral_code`. Sign-up with `?ref=CODE` sets `referred_by`; a customer can't refer themselves (same phone or device).
- When the referee makes their **first paid purchase** (not trial or voucher), both get the reward from settings: `referralRewardReferrer` and `referralRewardReferee` (default 30 min time pass each), and the referral becomes `rewarded`.
- Cap: `referralMaxPerMonth` per referrer (default 20).

### 11.4 Giveaway voucher batches

Voucher batch with `price_each = 0`, printed for events or social media. Reports show them as giveaways, not revenue.

---

## 12. Customer support

- **Help page (portal):** FAQ (accordion), WhatsApp support link (setting `supportWhatsapp`), opening hours, and the "Contact us" form.
- **Tickets:** the customer opens one with a category and message. For `payment`, the form lists their recent MoMo requests and Paystack transactions to attach, and a field for the MoMo transaction ID. Staff reply in the admin. The customer sees replies live (SSE) and gets an unread badge on the portal menu.
- **Live chat:** the same ticket thread shown as chat bubbles with realtime updates. The staff dashboard has a "Chat" inbox sorted by last message, with a sound and badge on new messages. Business hours: outside them, show "We reply from 08:00".
- **Quick actions from a ticket** (staff, depending on permissions): open the customer, credit a MoMo payment, gift time, resend OTP, reset password.
- **System messages** in the thread: "Payment GHS 10.00 credited by staff".
- **Auto-replies:** the first message in a `payment` ticket gets a system reply with the troubleshooting steps.
- **Announcements:** a banner on the portal home (info, promo or warning), with dates.
- **Rate limits:** 5 new tickets per customer per day, 30 messages per hour.

---

## 13. Customer portal (`/portal/`)

Mobile-first, works on small cheap phones and inside captive mini-browsers. No external resources. Light/dark/auto theme. English text in one `strings.ts` so it can be translated later.

| Page | Path | Content |
|---|---|---|
| Welcome | `/portal/` | Brand, announcement banner, plan cards with prices (happy-hour prices crossed out), buttons **Log in**, **Create account**, **I have a voucher**, **Free 15 minutes** (if trial on). Shows "You're connected ✓ — 42 min left" if this device already has a session. |
| Sign up | `/portal/signup` | Name, phone (with +233 hint; accept `024…` and convert), password ×2, optional email, referral code (pre-filled from `?ref`), terms checkbox. Optional OTP step if `requirePhoneVerify`. After sign-up, go to Home and offer the trial. |
| Log in | `/portal/login` | Phone + password, "Forgot password?" (OTP via the relay SMS, or "Ask staff for a reset code"). 5 wrong tries → 15 min lock per phone. |
| Home / account | `/portal/home` | Big status card: **Connected / Paused / Not connected**, with a live countdown (time) or data left with a progress bar, and buttons **Connect / Pause / Disconnect**. Below: **Wallet GHS 12.50** with a **Top up** button, **My passes** (each with what's left and its expiry), quick **Buy** shortcuts. |
| Buy | `/portal/plans` | Plan cards → bottom sheet: price breakdown, promo code box, "Pay GHS X from wallet", "Connect now" toggle. If the wallet is short: "Top up GHS Y and buy" (start a MoMo top-up for exactly the shortfall, then auto-buy when credited; store `pending_purchase_plan_id` on the MoMo request or Paystack transaction). |
| Top up | `/portal/topup` | Tabs **MoMo** · **Card / Paystack** · **Voucher**. MoMo flow as in §9.1, with a live waiting screen and a green success screen. Paystack redirect. Voucher code box. |
| Payment result | `/portal/paid` | For Paystack returns: spinner, then green or red. |
| Redeem | `/portal/redeem?code=` | Pre-filled voucher redeem (QR target). Works for guests (§9.3). |
| History | `/portal/history` | Tabs: **Money** (wallet_tx list) · **Usage** (sessions: date, device, duration, MB) · **Purchases**. |
| Devices | `/portal/devices` | Devices with label, last seen, active session; rename, disconnect, remove. Tip about private Wi-Fi address. |
| Deals | `/portal/deals` | §10.4. |
| Help | `/portal/help` | §12: FAQ, tickets list, new ticket, chat thread. |
| Profile | `/portal/profile` | Name, email, change password, theme, sign out, sign out everywhere, "Delete my account" (asks staff; doesn't delete money records). |

**Realtime:** `GET /api/portal/events` (SSE, auth by cookie) sends `wallet.changed`, `pass.changed`, `session.changed` (with `secondsLeft`, `bytesLeft`), `momo.status`, `paystack.status`, `ticket.message`, `warning` ("10 minutes left"). The countdown runs locally between events and resyncs on each event.

**Auth:** an httpOnly cookie `wh_c` (JWT, 30 days, sliding) plus a CSRF token header for POSTs (double-submit). Captive mini-browsers (iOS CNA) may not keep cookies after closing, so login must be fast.

---

## 14. Admin dashboard (`/admin/`)

Staff log in with username + password (cookie `wh_s`, 12 h). Permissions by role:

| Permission | owner | admin | cashier | support |
|---|---|---|---|---|
| dashboard.view, customers.view, sessions.view | ✓ | ✓ | ✓ | ✓ |
| sessions.kick, devices.block | ✓ | ✓ | ✓ | |
| wallet.topup (cash), vouchers.sell, shifts.use | ✓ | ✓ | ✓ | |
| momo.confirm | ✓ | ✓ | ✓ (optional PIN) | |
| wallet.adjust, passes.gift, purchases.refund | ✓ | ✓ | | |
| support.reply | ✓ | ✓ | ✓ | ✓ |
| plans.manage, promotions.manage, giveaways.manage, vouchers.create | ✓ | ✓ | | |
| reports.view | ✓ | ✓ | | |
| settings.manage, staff.manage, gateway.manage | ✓ | | | |

| Page | Content |
|---|---|
| **Dashboard** | Tiles: online now, revenue today (by method), new customers today, data used today, MoMo waiting count, open tickets. A live list of active sessions. Gateway health badge (driver, table OK, forwarding on, dnsmasq up, relay phone last seen). |
| **Live sessions** | Table: customer, device, IP/MAC, pass, time or data left, speed, started. Actions: **Kick**, **Pause**, **Add time**, **Block device**. Updates live. |
| **Customers** | Search by name or phone. Detail: wallet, passes, devices, sessions, payments, tickets, referrals, audit trail. Actions: add money (cash), Paystack QR, gift time or data, adjust (with reason), refund purchase, reset password (issue a 6-digit code valid 48 h), block or unblock, sign out everywhere, notes. |
| **Plans** | CRUD, ordering, badge, visibility, archive (never hard delete a plan with sales). Speed tiers. |
| **Promotions** | Promo codes (create, bulk-generate codes, usage stats), automatic promotions (happy hour etc.) with a preview "price now". |
| **Giveaways** | Campaigns, trial settings, referral settings and stats. |
| **Vouchers** | Batches, print sheet, CSV, void unused codes, record sale ("sold 20 cards, GHS 60 cash"). Code search. |
| **MoMo inbox** | Tabs: **Waiting**, **Unclaimed SMS**, **History**, **Outbox (SMS sent)**, **Relay phones** (pair, link, revoke, last seen). |
| **Paystack** | Transactions, status, fees, re-verify button. |
| **Support** | Ticket and chat inbox (filters: open, mine, payment), thread view with quick actions. FAQ editor. Announcements. |
| **Shifts** | Open or close a shift, cash count, MoMo count, differences. |
| **Reports** | Revenue by day, week or month and by method; plans sold; average spend per customer; data per day; peak hours heatmap; promotions cost (discounts given); vouchers sold vs redeemed. CSV export. |
| **Network** | Gateway status, mode, interfaces, DHCP leases, **Always allowed devices** (bypass MACs: owner phone, relay phone, printer), walled-garden domains, "Repair network" (re-init + reconcile), "Disconnect everyone". |
| **Settings** | §18.2 grouped: Business, Region & money, Portal, Sessions, MoMo, Paystack, SMS templates, Trial & referrals, Support, Security. |
| **Staff** | Users, roles, PINs, last login. |
| **Audit log** | Filterable list. |

---

## 15. API

Prefix `/api`. JSON. Errors are `{ error: { code, message } }` with friendly `message` text. All bodies are validated with zod.

**Public / captive**

```
GET  /api/captive                         RFC 8910 JSON for this client
GET  /api/public/config                   brand, currency, plans, promotions, trial on/off, announcements, support links
GET  /api/health                          { ok, db, gateway }
```

**Customer auth**

```
POST /api/portal/signup                   {fullName, phone, password, email?, referralCode?}
POST /api/portal/login                    {phone, password}
POST /api/portal/logout
POST /api/portal/otp/send                 {phone, purpose}       (via relay SMS outbox)
POST /api/portal/otp/verify               {phone, code, purpose, newPassword?}
POST /api/portal/reset-with-code          {phone, code, newPassword}   (staff-issued code)
GET  /api/portal/me                       profile + wallet + passes + device + session + unread counts
PATCH /api/portal/me                      {fullName?, email?}
POST /api/portal/me/password              {current, next}
```

**Customer: internet**

```
GET  /api/portal/device                   this device (by IP→MAC), its session, captive?
POST /api/portal/connect                  {passId?}
POST /api/portal/pause
POST /api/portal/disconnect               {sessionId?}  (another device's session too)
GET  /api/portal/devices      PATCH /api/portal/devices/:id {label}      DELETE /api/portal/devices/:id
GET  /api/portal/passes
GET  /api/portal/sessions?limit=
```

**Customer: buying and money**

```
GET  /api/portal/plans
GET  /api/portal/quote?planId=&code=
POST /api/portal/purchases                {planId, code?, connectNow?}
GET  /api/portal/wallet/tx?limit=
GET  /api/portal/momo                     accounts, networks, steps, min/max, my pending requests
POST /api/portal/momo                     {amount, network, buyPlanId?}
POST /api/portal/momo/:id/cancel
POST /api/portal/momo/:id/txn             {txnId}   claim with transaction ID
POST /api/portal/topups/paystack          {amount, buyPlanId?}
GET  /api/portal/topups/paystack/:ref
POST /api/portal/vouchers/redeem          {code}
POST /api/portal/guest/voucher            {code}   (no account; creates guest + connects)
POST /api/portal/guest/save               {fullName, phone, password}
```

**Customer: deals, giveaways, support**

```
GET  /api/portal/deals
POST /api/portal/promo/redeem             {code}   (free_plan / wallet_credit codes)
POST /api/portal/trial
POST /api/portal/giveaways/:id/claim
GET  /api/portal/referrals
GET  /api/portal/faq
GET  /api/portal/tickets      POST /api/portal/tickets {category, subject, body, momoRequestId?, paystackRef?, txnId?}
GET  /api/portal/tickets/:id  POST /api/portal/tickets/:id/messages {body}
GET  /api/portal/events                   SSE
```

**MoMo relay** (§9.1): `/api/momo/relay/pair`, `/api/momo/sms`, `/api/momo/relay/ping`, `/api/momo/relay/outbox`, `/api/momo/relay/outbox/:id`.

**Paystack:** `POST /api/paystack/webhook` (raw body for HMAC).

**Admin** (`/api/admin/*`, staff cookie and permission per route): `auth/login`, `auth/logout`, `me`, `dashboard`, `sessions` (+ `/:id/kick|pause|add-time`), `customers` (+ `/:id`, `/:id/wallet/topup`, `/:id/wallet/adjust`, `/:id/passes/gift`, `/:id/passes/:passId/adjust`, `/:id/purchases/:pid/refund`, `/:id/reset-code`, `/:id/block`, `/:id/signout`, `/:id/paystack-qr`), `devices` (+ `/:id/block`, bypass list), `plans`, `speed-tiers`, `promo-codes` (+ `/bulk`), `promotions`, `giveaways`, `trial-settings`, `vouchers/batches` (+ `/:id/print`, `/:id/csv`, `/:id/void`, `/:id/sold`), `vouchers/search`, `momo/requests` (+ `/:id/confirm|reject`), `momo/sms` (+ `/:id/credit|ignore`), `momo/relay` (+ `/code`, `/link`, `/:id` DELETE), `momo/outbox` (+ `/test`, `/:id/retry`), `paystack/tx` (+ `/:ref/verify`), `paystack/test`, `tickets` (+ `/:id`, `/:id/messages`, `/:id/assign`, `/:id/close`), `faq`, `announcements`, `shifts` (+ `/open`, `/:id/close`), `reports/*`, `network/status`, `network/repair`, `network/disconnect-all`, `network/leases`, `settings`, `staff`, `audit`, `events` (SSE).

**Dev only** (`NODE_ENV !== 'production'` and `GATEWAY_DRIVER=mock`): `POST /api/dev/traffic {ip, up, down}`, `POST /api/dev/clock {advanceSeconds}`, `POST /api/dev/sms {from, body}` (simulates the relay).

---

## 16. Background jobs (`jobs/scheduler.ts`)

Simple in-process `setInterval` with a "running" guard (no overlap) and error logging. One app instance only, which is fine for a home server.

| Job | Every | Does |
|---|---|---|
| `tick` | 15 s | §8.3 usage, cut-offs, idle pause, warnings |
| `expiry` | 60 s | §8.5 |
| `reconcile` | 5 min (and at start) | §8.6 |
| `paystack-sweep` | 60 s | verify pending Paystack transactions |
| `momo-late-match` | 60 s | retry matching unmatched SMS against new requests (customer started the request after paying) |
| `outbox-cleanup` | 10 min | expire SMS older than 12 h, mark stuck `sending` after 5 min as failed |
| `daily-stats` | 00:05 local | roll up `daily_stats` (revenue, users, bytes) for fast reports |
| `cleanup` | 03:30 local | delete expired OTPs, auth tokens and old audit rows beyond the retention setting |

The clock is injectable (`ctx.clock.now()`) so tests can advance time.

---

## 17. Security

- **Passwords:** scrypt (N=16384, r=8, p=1, 64-byte key, 16-byte salt). Minimum 6 characters for customers and 8 for staff. Lock after 5 failures for 15 min (per phone/username and per IP).
- **Tokens:** JWT signed with `JWT_SECRET`, in httpOnly `SameSite=Lax` cookies; `Secure` when served over HTTPS. Token IDs are stored hashed so they can be revoked.
- **CSRF:** double-submit token header on all state-changing portal and admin requests.
- **Rate limits** (`@fastify/rate-limit`): login/OTP/voucher/promo endpoints strict; global 300 req/min per IP.
- **Customer isolation:** every portal query is filtered by `customer_id` from the token. Never take a customer ID from the body.
- **Admin from customer network:** `/admin` and `/api/admin/*` are **refused from `LAN_SUBNET`** unless the device is in the bypass list (setting `adminFromCustomerWifi` default false). Staff use the owner network, the server itself or the Cloudflare address with Cloudflare Access.
- **Secrets at rest:** Paystack secret key encrypted with AES-256-GCM using a key derived from `APP_SECRET`.
- **Relay tokens:** 32 random bytes, stored as sha256, revocable.
- **Gateway:** strict validation and no shell (CLAUDE.md rule 5). The app container has only `NET_ADMIN` (and `NET_RAW` for conntrack), not `privileged`.
- **Client isolation on Wi-Fi:** hostapd `ap_isolate=1` (Mode A with USB Wi-Fi). For an external AP, tell the owner to enable "AP isolation / client isolation" so customers can't attack each other.
- **Headers:** `@fastify/helmet` with a CSP that allows only self (and the Paystack redirect is a top-level navigation, so no CSP exception is needed).
- **Logging:** pino; never log passwords, tokens, OTP codes or full SMS bodies at info level.
- **Backups** contain personal data: the backup folder is chmod 700.

---

## 18. Configuration

### 18.1 `.env` (`.env.example` must contain all of these with comments)

```dotenv
# ── Core ──
NODE_ENV=production
APP_PORT=8080                       # the portal/API port (nft redirects :80 here)
APP_SECRET=change-me-64-random-chars    # encryption of stored secrets
JWT_SECRET=change-me-64-random-chars
DATABASE_URL=postgres://wifihub:${POSTGRES_PASSWORD}@127.0.0.1:5432/wifihub
POSTGRES_PASSWORD=change-me
TZ=Africa/Accra
ADMIN_USERNAME=admin                # first owner account, created if no staff exists
ADMIN_PASSWORD=                     # required on first start

# ── Gateway ──
GATEWAY_DRIVER=nftables             # nftables | mock
GATEWAY_MODE=A                      # A = two interfaces, B = single interface (on a stick)
WAN_IF=enp1s0
LAN_IF=wlan0                        # same as WAN_IF in Mode B
LAN_SUBNET=10.50.0.0/24
PORTAL_IP=10.50.0.1
PORTAL_HOST=wifi.hub
PRIVATE_BLOCK=192.168.8.0/24        # owner network customers must never reach (Mode A); empty in Mode B
DNSMASQ_LEASES=/var/lib/misc/dnsmasq.leases
GARDEN_CONF=/etc/dnsmasq.d/wifihub-garden.conf

# ── Public access (optional) ──
PUBLIC_URL=                         # e.g. https://wifi.example.com (Cloudflare Tunnel), needed for Paystack webhooks
CLOUDFLARE_TUNNEL_TOKEN=
PAYSTACK_FALLBACK_EMAIL_DOMAIN=     # e.g. example.com → 233241234567@example.com when a customer has no email

# ── Backups ──
BACKUP_PATH=./backups
BACKUP_HOUR=3
BACKUP_KEEP_DAYS=14
```

### 18.2 Settings (database, edited in Admin → Settings, zod schema with these defaults)

| Group | Keys (default) |
|---|---|
| Business | `brandName` ("ZENAA WiFi"), `ssidName` ("ZENAA WiFi"), `logoUrl`, `supportWhatsapp`, `supportPhone`, `openingHoursText`, `termsText` |
| Region | `countryCode` ("GH"), `currencyCode` ("GHS"), `currencyMinorUnits` (2), `timezone` ("Africa/Accra"), `phonePrefix` ("+233") |
| Portal | `selfSignup` (true), `requirePhoneVerify` (false), `voucherGuestMode` (true), `showPlansBeforeLogin` (true) |
| Sessions | `idleMinutes` (5), `autoContinue` (true), `warnMinutes` ([10, 2]), `warnMegabytes` ([100, 20]), `smsLowBalance` (false), `adminFromCustomerWifi` (false) |
| Wallet | `walletCashOut` (false), `requireOpenShift` (false) |
| MoMo | `momoTopupEnabled` (true), `momoAccounts` ([] of `{network, number, name}`), `momoNetworks` (["MTN MoMo","Telecel Cash","AT Money"]), `momoSteps` (per network text), `momoMinTopup` (100), `momoMaxTopup` (200000), `momoAutoConfirm` (true), `momoAutoMax` (30000), `momoAutoHoursOn` (false), `momoAutoFrom` ("06:00"), `momoAutoTo` ("23:00"), `momoSmsSenders` (["MobileMoney","MTN MoMo","MTNMoMo","T-CASH","TelecelCash","Telecel","VodaCash","AT Money","AirtelTigo"]) |
| Paystack | `paystackEnabled` (false), `paystackPublicKey`, `paystackSecretKeyEnc`, `paystackChannels` (["card","mobile_money"]), `walledGardenPaystack` (false), `walledGardenExtra` ([]) |
| SMS | `smsOn` (true), `smsTemplateTopup` ("Hi {name}, {amount} added to your {brand} wallet. Balance: {balance}."), `smsTemplateOtp` ("{brand} code: {code}. Valid 10 minutes."), `smsTemplateLow` ("{brand}: {left} left on your Wi-Fi. Top up at {portal}."), `smsTemplateExpired` |
| Promotions | `stackPromotions` (false) |
| Trial | `trialEnabled` (true), `trialMinutes` (15), `trialRequiresAccount` (true), `trialRequiresVerifiedPhone` (false), `trialSpeedTierId` (null) |
| Referrals | `referralEnabled` (true), `referralRewardReferrer` ({kind:"time", seconds:1800}), `referralRewardReferee` (same), `referralMaxPerMonth` (20) |
| Support | `supportHoursFrom` ("08:00"), `supportHoursTo` ("22:00"), `ticketAutoReplyPayment` (text) |
| Security | `requirePinFor` ([] of `momo_confirm`, `wallet_adjust`, `refund`) |

### 18.3 Seed data (first start, only if there are no plans)

| Plan | Kind | Price | Limits |
|---|---|---|---|
| 30 Minutes | time | GHS 1.00 | 1,800 s, use within 7 days |
| 1 Hour | time | GHS 2.00 | 3,600 s, use within 30 days |
| 3 Hours | time | GHS 5.00 | 10,800 s, use within 30 days |
| Day Pass | validity | GHS 8.00 | 24 h from first use |
| Week Unlimited | validity | GHS 40.00 | 7 days, 2 devices |
| 2 GB Bundle | data | GHS 10.00 | 2 GiB, 7 days |

These are examples; the owner sets real prices. Also seed 3 FAQ entries and one welcome announcement.

---

## 19. Docker and host setup

### 19.1 `docker-compose.yml`

```yaml
name: wifihub
services:
  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: wifihub
      POSTGRES_USER: wifihub
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD in .env}
      TZ: ${TZ:-Africa/Accra}
    volumes: [pgdata:/var/lib/postgresql/data]
    ports: ["127.0.0.1:5432:5432"]          # host-network app reaches it on localhost only
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U wifihub -d wifihub"]
      interval: 10s
      retries: 10

  app:
    build: .
    image: wifihub:latest
    restart: unless-stopped
    network_mode: host                      # sees real client IPs, manages host nftables
    cap_add: [NET_ADMIN, NET_RAW]
    depends_on: { db: { condition: service_healthy } }
    env_file: .env
    volumes:
      - /var/lib/misc:/var/lib/misc:ro      # dnsmasq leases
      - /etc/dnsmasq.d:/etc/dnsmasq.d       # walled-garden file written by the app
      - /run/wifihub:/run/wifihub           # helper socket (§19.3)
    logging: { driver: json-file, options: { max-size: "10m", max-file: "5" } }

  backup:
    image: postgres:16-alpine
    restart: unless-stopped
    depends_on: { db: { condition: service_healthy } }
    environment:
      PGHOST: db
      PGUSER: wifihub
      PGPASSWORD: ${POSTGRES_PASSWORD}
      PGDATABASE: wifihub
      BACKUP_HOUR: ${BACKUP_HOUR:-3}
      BACKUP_KEEP_DAYS: ${BACKUP_KEEP_DAYS:-14}
      TZ: ${TZ:-Africa/Accra}
    volumes:
      - ${BACKUP_PATH:-./backups}:/backups
      - ./infra/backup/backup-loop.sh:/backup-loop.sh:ro
    entrypoint: ["/bin/sh", "/backup-loop.sh"]

  cloudflared:
    profiles: ["public"]                    # docker compose --profile public up -d
    image: cloudflare/cloudflared:2024.12.2
    restart: unless-stopped
    network_mode: host
    command: tunnel --no-autoupdate run --token ${CLOUDFLARE_TUNNEL_TOKEN}

volumes: { pgdata: {} }
```

**Dev compose override** (`docker-compose.dev.yml`): app with `GATEWAY_DRIVER=mock`, normal bridge network, `ports: ["8080:8080"]`, so it runs on Windows or Mac too.

**Dockerfile:** stage 1 `node:22-bookworm-slim` builds web and server (`npm ci`, `npm run build`). Stage 2 `node:22-bookworm-slim` + `apt-get install -y --no-install-recommends nftables conntrack iproute2`, copies `server/dist`, `server/drizzle`, `web/dist` and production `node_modules`. It runs migrations at start, then `node dist/index.js`. Run as root inside the container (needed for nft with `NET_ADMIN`); no other privileges.

### 19.2 Host setup script (`infra/gateway/setup.sh`, Debian/Ubuntu)

Interactive with sensible defaults, idempotent, and it prints every change. Steps:

1. Check root, OS (Debian 12 / Ubuntu 22.04+), list interfaces (`ip -br link`), ask for `GATEWAY_MODE`, `WAN_IF`, `LAN_IF`, SSID and password (blank = open) if `LAN_IF` is wireless, and subnet.
2. `apt-get install -y dnsmasq nftables conntrack iproute2` (+ `hostapd` and `iw` if wireless). Check that `iw list` shows `AP` in "Supported interface modes"; if not, stop with "This Wi-Fi adapter can't be an access point".
3. Mode A: give `LAN_IF` the static IP `PORTAL_IP/24` with systemd-networkd (or netplan on Ubuntu); tell NetworkManager to leave it unmanaged (`/etc/NetworkManager/conf.d/99-wifihub.conf`: `unmanaged-devices=interface-name:<LAN_IF>`).
   Mode B: give the server a static IP on the Huawei subnet and keep its default route via the Huawei; print "Now turn OFF DHCP on the Huawei router (Settings → LAN/DHCP)".
4. `/etc/sysctl.d/99-wifihub.conf`: `net.ipv4.ip_forward=1`, `net.netfilter.nf_conntrack_acct=1`, `net.ipv6.conf.<LAN_IF>.disable_ipv6=1`.
5. dnsmasq `/etc/dnsmasq.d/wifihub.conf` from the template:
   ```
   interface=<LAN_IF>
   bind-dynamic
   dhcp-range=<start>,<end>,255.255.255.0,1h
   dhcp-option=option:router,<PORTAL_IP>
   dhcp-option=option:dns-server,<PORTAL_IP>
   dhcp-option=114,"http://<PORTAL_HOST>/api/captive"
   address=/<PORTAL_HOST>/<PORTAL_IP>
   domain-needed
   bogus-priv
   no-resolv
   server=1.1.1.1
   server=8.8.8.8
   cache-size=2000
   ```
   Make sure dnsmasq doesn't fight `systemd-resolved` on port 53 (`bind-dynamic` + `interface=` only; or set `DNSStubListener=no`).
6. hostapd (wireless `LAN_IF`): `ssid`, `hw_mode=g`, `channel=6`, `ieee80211n=1`, `wmm_enabled=1`, `ap_isolate=1`, `country_code=GH`, optional WPA2. Enable `hostapd`.
7. Install `wifihub-helper` (§19.3) as a systemd service.
8. Write the chosen values into `.env` (gateway section).
9. Print a checklist: "Start the app: `docker compose up -d --build`", "Connect a phone to <SSID>; it should show the sign-in page", "Add your own phone and the MoMo relay phone to Admin → Network → Always allowed".

Also provide `uninstall.sh` (removes configs, deletes the nft table, re-enables NetworkManager on LAN_IF) and `status.sh` (interfaces, forwarding, `nft list table inet wifihub | head`, dnsmasq and hostapd status, lease count).

### 19.3 Host helper (`wifihub-helper`)

The app container must not get broad host control. A tiny root service on the host listens on a Unix socket `/run/wifihub/helper.sock` and accepts **only** these fixed commands: `restart-dnsmasq`, `status`. Write it as ~60 lines of Node or Python with no arguments passed through. The app uses it after writing the walled-garden file. If the socket is missing, the admin shows a manual instruction instead.

---

## 20. Testing

`server/test/` with vitest + PGlite (fresh database per file, run migrations) + mock gateway + fake clock. Coverage that **must** exist:

- **Money:** purchase with enough and not enough money; concurrent double purchase can't overdraw (two parallel requests, one fails); refund rules; ledger sums equal `wallet_balance` after every test (a helper asserts this).
- **Quote:** happy hour inside and outside the window (including across midnight); promo code limits (max uses under concurrency, per customer, first purchase only); no stacking vs stacking.
- **Sessions:** connect allows the gateway with the correct timeout for each pass kind; tick deducts time; data deduction from counters; cut-off at zero calls `revoke`; auto-continue to the next pass; idle pause and refund; max devices; reconnect resumes a paused session; reconcile re-adds missing entries and removes orphans.
- **MoMo:** parser samples (§9.1.2); untrusted sender; duplicate transaction ID; match by code, by transaction ID, by payer phone; auto-credit limits and hours; staff confirm; late match job.
- **Paystack:** initialise (mock `fetch`), verify success credits once even when called 4 times concurrently; amount mismatch fails; webhook signature valid and invalid.
- **Vouchers:** redeem once; concurrent redeem of the same code; rate limit; guest flow.
- **Giveaways:** slots under concurrency; trial once per device and once per customer per day.
- **Security:** portal routes can't read another customer's data; admin routes are refused from `LAN_SUBNET`; permission matrix.
- **Gateway nftables driver:** unit-test `ruleset.ts` output (snapshot) and the validation functions (bad IP or MAC or seconds are rejected). Integration test against real nft only when `NFT_INTEGRATION=1` (runs in a network namespace via `unshare -rn`).

Web: at least a smoke test that the portal and admin build, and Playwright e2e in mock mode for: sign up → MoMo top-up (simulate SMS through the dev endpoint) → buy 1 hour → connect → advance clock → cut off → portal shows "Time's up".

---

## 21. Build phases and acceptance checks

Do these in order. Each phase ends with typecheck + tests + build passing, a commit, and an update to DEVELOPMENT_LOG.md.

### Phase 0: Scaffold

- Folder layout (§4), package.json scripts, TypeScript strict, ESLint/Prettier, vitest + PGlite helper, drizzle config, `.env.example`, Dockerfile, docker-compose (+ dev override), `config.ts`, `context.ts`, error handling, health route, settings service with zod defaults, audit service, seed.
- **Check:** `docker compose -f docker-compose.yml -f docker-compose.dev.yml up` serves `/api/health` → `{ok:true}`; `npm test` passes.

### Phase 1: Accounts, plans, wallet, passes, sessions, cut-off, vouchers (the core)

- Customer sign-up, login, logout, me; staff login, roles and permissions; first owner from env.
- Plans CRUD + seed; wallet ledger; cash top-up by staff; purchase; passes.
- Gateway driver interface, **mock driver**, **nftables driver** (ruleset, allow/extend/revoke/counters/listAllowed/bypass/lookupMac/status) with validation and an nft call queue.
- Captive redirect hook and `/api/captive`; connect, pause and disconnect; tick, expiry and reconcile jobs; SSE.
- Vouchers: batches, codes, print sheet with QR, redeem, guest mode; Wi-Fi poster QR.
- Portal pages: Welcome, Sign up, Log in, Home (status, countdown, wallet, passes), Buy, Top up (voucher tab only for now), History, Devices, Profile.
- Admin pages: Login, Dashboard, Live sessions, Customers (detail, add cash, gift, kick), Plans, Vouchers, Network (status, bypass list, repair), Staff, Audit.
- **Check (mock mode):** sign up → staff adds GHS 10 cash → buy "1 Hour" → connect → mock gateway shows the device allowed with timeout 3600 → advance the clock 3600 s → session ended `time_up`, gateway revoked, portal shows "Time's up". Voucher: print a batch, redeem one as a guest, get connected.

### Phase 2: Payments (MoMo direct + Paystack + SMS outbox)

- MoMo requests, parser, relay pairing, token API (ZENAA MoMo compatible), matching, auto-credit rules, MoMo inbox (waiting, unclaimed, history), transaction-ID claim, late match job.
- SMS outbox + templates; OTP for sign-up verification and password reset over the outbox; staff reset codes.
- Paystack: settings (encrypted key), initialise, return page, polling, webhook, sweep, admin list, counter QR, test connection.
- "Top up the shortfall and buy" flow.
- Owner doc `docs/MOMO-RELAY.md`: how to pair the ZENAA MoMo app or any SMS-forwarder app, and how to allow SMS permission (copy the structure of ZENAA doc 18).
- **Check:** `POST /api/dev/sms` with the MTN sample and the right code → wallet credited once, portal turns green, thank-you SMS queued; same SMS again → ignored; Paystack test mode payment credits once.

### Phase 3: Discounts, promotions, giveaways, referrals

- Promo codes (all effects, bulk generate), automatic promotions, `quote()`, Deals page, crossed-out prices, happy-hour banner.
- Free daily trial, giveaway campaigns, referrals, giveaway voucher batches.
- Admin: Promotions, Giveaways pages with stats.
- **Check:** all §20 promotion and giveaway tests; a happy hour at 07:00 shows the discounted price and charges it.

### Phase 4: Customer support

- Tickets, chat thread with SSE, unread badges, auto-reply, quick actions, FAQ, announcements, WhatsApp link, rate limits.
- **Check:** customer opens a payment ticket with a transaction ID → staff sees it live, credits from the ticket → the system message appears in the thread.

### Phase 5: Reports, shifts, polish

- Shifts with cash and MoMo counts; reports (§14) + `daily_stats` job + CSV export.
- Portal polish: light/dark theme, loading and error states, offline banner, accessibility (labels, contrast, 44 px tap targets), bundle size check (< 250 KB gzipped).
- **Check:** reports match the ledger totals in tests; Lighthouse mobile score ≥ 90 on the portal in mock mode.

### Phase 6: Real network, speed tiers, hardening

- `infra/gateway/setup.sh`, `uninstall.sh`, `status.sh`, templates, `wifihub-helper`.
- Walled-garden file generation; speed tiers (§6.6); backups; `docs/SETUP.md` for the owner (step by step, with screenshots placeholders, for Mode A with a USB Wi-Fi adapter, Mode A with an old router as AP, and Mode B).
- Security review against §17.
- **Check on the real server:** phone joins the SSID → sign-in popup appears → buy with MoMo → internet works → at zero it stops within 15 s and open downloads die → reboot the server → paying customers are back online automatically after the app starts.

---

## 22. Operations and troubleshooting (put this into `docs/SETUP.md`)

| Symptom | Check |
|---|---|
| No sign-in popup | Is the device getting a 10.50.0.x address (DHCP)? `status.sh`. Open `http://wifi.hub` manually. The phone may have "Private DNS" set to a provider, so tell customers to set it to Automatic. |
| Paid but no internet | Admin → Live sessions: is the session active? Network → status: table present, forwarding on? Press **Repair network**. |
| Internet works without paying | Mode B bypass (static gateway): move to Mode A. Or the device is in the bypass list. |
| Customers lose time while away | Check `idleMinutes`; background apps keep traffic flowing, which counts as use. |
| MoMo not auto-credited | Relay phone last seen? Sender in the trusted list? See MOMO-RELAY.md. |
| Paystack page won't load before login | Enable the walled garden for Paystack, or tell customers to use MoMo direct. |
| Slow Wi-Fi with many users | USB Wi-Fi adapters handle ~10–20 users; use an old router or a proper AP in AP mode on `LAN_IF`. |

Updates: `git pull && docker compose up -d --build` (migrations run at start). Backups: `./backups/*.sql.gz`; restore with `gunzip -c file | docker compose exec -T db psql -U wifihub wifihub`.

---

## 23. Legal notes (show in SETUP.md)

- Reselling an internet connection may be against the terms of a home internet plan. Check with the ISP and get a business plan if needed.
- In Ghana, offering public internet access commercially may need authorisation from the National Communications Authority (NCA). Check before charging the public.
- Customer data (names, phone numbers, usage) is personal data under Ghana's Data Protection Act 2012 (Act 843): keep it secure, collect only what's needed, and don't share it.
- Keep payment records for tax purposes.
