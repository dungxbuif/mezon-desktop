# Code change reviews

Incremental, evidence-backed review packages for the `mezonai/mezon-desktop` repository, produced by the `track-codebase-evolution` workflow.

Each review covers the commits between the last reviewed commit and the current revision, grouped by capability/domain, with deep reports for Critical and Significant changes.

## Reviews

| Review                                                             | Commit range                                                     | Main themes                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Quality |
| ------------------------------------------------------------------ | ---------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------- |
| [2026-07-31_5a161c5-2c8c046](2026-07-31_5a161c5-2c8c046/README.md) | `5a161c5` (PR #92, 2026-07-15) → `2c8c046` (PR #178, 2026-07-30) | 8 new crates (canvas, mcp, webview, stream, theme, cli, widgets, mmn-client); HTTP transport fallback + token renewal; MMN wallet; MCP AI server + CLI; store domain expansion (15+ modules, ClanRoles breaking change); voice/stream channel + screen share; clan admin (roles, permissions, audit); canvas editor; channel mini-app webview; message actions + composer; UI primitives (TextArea, widgets, theme); notification settings, inbox, profile, media channel, gotify | PASS    |

## How to read a review

1. Start with the review's `README.md` — orientation, 5-minute summary, change-group table, reading paths.
2. `evidence.md` — git identity, diff totals, churn-by-crate, key commits.
3. `coverage.md` — mapping of all commits and changed paths to change groups.
4. Topic reports (one per Critical/Significant group) — Change, Before, After, Mechanism, Motivation, Impact, Trade-offs, Verification, Source-reading guide, Learning, Unknowns.
5. `quality-review.md` — skeptical self-review and PASS/INCOMPLETE decision.

## State

The last-reviewed commit is tracked in `.sync-state.json`. The next review will cover commits from that point forward.
