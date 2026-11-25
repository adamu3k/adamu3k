# WiFi Hub: instructions for Claude Code

This project is a prepaid Wi-Fi billing system that runs on a Linux home server.
Customers join the Wi-Fi, sign in on a captive portal, top up a wallet (MoMo, Paystack, voucher, cash), buy time or data, and are cut off automatically when it runs out.

**The full specification is `docs/BLUEPRINT.md`. Read it completely before writing any code, and follow it.**

## How to work

- Build in the phases listed in BLUEPRINT.md §21, in order. Finish a phase (code, migrations, tests, docs) before starting the next one, and tell the user which acceptance checks pass.
- After each phase: `npm run typecheck`, `npm test` (server) and `npm run build` (web) must pass. Commit with a clear message.
- If the blueprint is unclear, choose the simplest option that keeps the rules in §8 (money and time) and §17 (security), write the decision in `docs/DECISIONS.md`, and carry on. Ask the user only about things only they can decide (prices, phone numbers, brand name).
- Keep a short `DEVELOPMENT_LOG.md`: one entry per phase with what was built and anything left to do.

## Non-negotiable rules

1. **Money is integers in minor units** (pesewas: GHS 1.00 = 100). Time is integer seconds. Data is integer bytes (`bigint`). No floats for money, time or data.
2. **Every balance change goes through the wallet ledger or pass ledger** in one database transaction, with an audit row. Never update a balance column directly anywhere else.
3. **Never trust the browser about payments.** Credit only after a server-side check (Paystack verify API, or a trusted MoMo SMS match, or a staff confirmation). Each external transaction ID is credited at most once (unique index).
4. **The gateway fails closed.** Customers have no internet unless the gateway has an allow entry for them, and every allow entry has a timeout so it expires even if the app crashes.
5. **Gateway commands never go through a shell.** Use `execFile`/`spawn` with argument arrays, or `nft -f -` with a script built only from values that passed strict IP/MAC/integer validation.
6. **The portal must work inside captive-portal mini-browsers:** no external CDNs, fonts or scripts. Everything is self-hosted, and the portal bundle is small (target < 250 KB gzipped).
7. **No secrets in git.** Everything secret comes from `.env` (see `.env.example`).
8. Default region is **Ghana**: currency GHS (2 minor units), time zone `Africa/Accra`, phone prefix `+233`, networks MTN MoMo / Telecel Cash / AT Money. All of these are settings, not hard-coded.

## Commands (once scaffolded)

```bash
docker compose up -d --build                 # production on the home server
cd server && npm run dev                     # API in dev (GATEWAY_DRIVER=mock)
cd web && npm run dev                        # portal + admin with hot reload
cd server && npm test                        # vitest (uses PGlite, no Docker needed)
sudo ./infra/gateway/setup.sh                # one-time host network setup (Linux only)
```
