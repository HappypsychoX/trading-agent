# Changelog — trading-agent

All notable changes to the `trading-agent` skill. Format follows [Keep a Changelog](https://keepachangelog.com/); versions match the `[vX.Y.Z]` tag in `SKILL.md` and `plugin.json` `version`.

## [4.1.0] — 2026-08-20
- **Added:** Fractional / dollar-based position sizing. The agent may now size entries in fractional shares or a dollar notional instead of rounding to whole shares; these route as regular-hours market orders (Robinhood's only fractional path, no limit-price protection), while whole-share entries keep the limit-order default. A new SHARE_GRANULARITY note documents the tradeoff.
- **Added:** Read-only use of a connected Webull MCP for supplementary market data (snapshots, analyst ratings, financials, capital flow, technicals, industry comparison, earnings) and as a candidate source (gainers/losers, most-active, high-dividend, sectors). Optional — degrades gracefully if not connected. All execution and account truth remain on Robinhood.
- **Changed:** Clarified that a standing protective order can only cover whole shares, so protection-worthy theses should be sized to whole shares; fractional-only positions carry no standing stop.
- **Added:** Trigger phrases for fractional/partial shares, dollar-based orders, and Webull market data.

## [4.0.0] — 2026-08-01
- **Changed:** The cross-session note `position-notes.md` now lives in the **same directory as `trading-config.json`** instead of a separate connected notes folder — one connected folder holds both config and note, so there is nothing extra to wire up. Existing setups with a separate notes folder should move `position-notes.md` next to their config (or the first post-upgrade session will start a fresh note).

## [3.0.0] — 2026-08-01
- **Changed:** Externalized every environment-specific value into a single runtime config file, `trading-config.json`, located via the connected folder (suggested `%LOCALAPPDATA%/trading-agent`) — no hardcoded repo owner/name, file path, or account scope remains in the skill body.
- **Changed:** The GitHub token, dashboard repo (`owner`/`repo`/`branch`), risk-parameter path, and account scope are all read from that config file; the risk-parameter fetch composes its URL from those values.
- **Added:** A sanitized `trading-config.example.json` template (token and repo blanked) to copy and fill in; the real config lives outside the repo and is git-ignored.

## [2.0.1] — 2026-08-01
- **Fixed:** Instruction and wording cleanup in the skill body; no behavioral change to trading logic, protective orders, horizon bias, the leveraged-ETF screen, or risk-parameter sourcing.

## [2.0.0] — 2026-08-01
- **Changed:** Baseline under the versioning convention; renamed from the legacy `trading-skill-v4` to `trading-agent`.
- **Changed:** Point GitHub-backed infra at the `HappypsychoX/Trading-Dashboard` repo.
- **Changed:** Locate secrets via the connected folder instead of a hardcoded machine-specific path.
