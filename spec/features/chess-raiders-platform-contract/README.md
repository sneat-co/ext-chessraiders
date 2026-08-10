---
format: https://specscore.md/feature-specification
status: Implementing
---

# Feature: Chess Raiders platform contract

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/sneat-co/ext-chessraiders/spec/features/chess-raiders-platform-contract?op=explore) | [Edit](https://specscore.studio/app/github.com/sneat-co/ext-chessraiders/spec/features/chess-raiders-platform-contract?op=edit) | [Ask question](https://specscore.studio/app/github.com/sneat-co/ext-chessraiders/spec/features/chess-raiders-platform-contract?op=ask) | [Request change](https://specscore.studio/app/github.com/sneat-co/ext-chessraiders/spec/features/chess-raiders-platform-contract?op=request-change) |
**Status:** Implementing
**Source Ideas:** —

## Summary

Public Go contract (types, interfaces, errors — no gameplay, no engine logic) for the lobby/social-shell boundary, external identity, and portal session binding between Chess Raiders and its hosts.

## Problem

Chess Raiders' engine and application facade (`sneat-co/chessraiders`) are
private, on purpose (`server-go/chess` must never appear in a public tree).
But three integration seams already exist today with no shared, public
vocabulary behind them:

- **Lobby/social-shell**: `sneat-go`'s `pkg/modules/chess` and the private
  `sneat-bots` repo's `extensions/chess/bot/cmds4chessbot` each independently
  declare their own `View`/`Player`/`CaptureMode`/error types for the same
  concept, bridged only by a hand-maintained translation table
  (`mapFacadeError` in `bot_service.go`). That is the exact "one list beside
  the thing it mirrors" anti-pattern `chessraiders/AGENTS.md` warns about
  generally, just not yet named as a problem for this seam specifically.
- **External identity** (decisions 0011–0013 in `sneat-co/chessraiders`):
  decision 0012 already specifies an `ExternalIdentity` /
  `ExternalIdentityVerifier` shape in prose, for CrazyGames/itch.io portal
  login federation, but it exists only as private prose + ad hoc private Go
  today (`sneat-go`'s `web_api.go` resolves it directly against
  chessraiders' private `api4chess.Config` hooks).
- **Portal session binding** (decisions 0010–0011): a distribution portal's
  own room/session concept (CrazyGames' `roomId`) needs to bind to a match,
  with no typed seam for that either.

None of this required a *new* capability — the private repos already do all
three, just without a shared, credential-free vocabulary a third party (or a
sibling extension) could import.

## Behavior

`backend/contract4chessraiders` (Go, stdlib-only, matching the platform's
`contract4competios` precedent) declares:

- `Side`, `CaptureMode`, `Lifecycle`, `Player`, `LobbySeat`, `LobbyView` — the
  deduplicated, public version of the lobby/social-shell vocabulary.
- `MatchLobbyApplication` — create/join/leave/ready/start/get, with its error
  set (`ErrMatchNotFound`, `ErrNotInLobby`, `ErrNotCreator`,
  `ErrTeamsNotViable`, `ErrNotAllReady`, ...).
- `IdentityProvider`, `ExternalIdentity`, `ExternalIdentityVerifier`,
  `IdentityLinkApplication` — the typed form of decision 0012.
- `PortalSessionRequest`, `PortalSession`, `PortalSessionApplication` — the
  portal room/session binding seam from decisions 0010–0011.

`sneat-go` (the host) wires four adapter constructors against these
interfaces from `provideChessInternal()` in `pkg/modules/chess`:
`NewMatchLobbyApplication`, `NewIdentityLinkApplication`,
`NewPortalSessionApplication`, and `NewCompetitionGameLauncher` (the last one
returns the *already-public* `contract4competios.GameLauncher` directly —
reuse, not a new type — formalising the existing
`facade4chess.NewCompetiosAdapter` composition as one of the four).

Gameplay, board state, and anything derived from `server-go/chess` stay out
of this contract entirely; see the repository README's "Scope" section for
the full exclusion list and reasoning per exclusion.

## Acceptance Criteria

- AC-1: Given a Go program with no chessraiders/sneat-go/sneat-bots
  credentials, when it runs `go get github.com/sneat-co/ext-chessraiders/backend`,
  then the module resolves and builds with zero non-stdlib dependencies
  (`go.mod` has no `require` block, no `go.sum`; enforced in CI by the
  `no_dependencies` job).
- AC-2: Given `contract4chessraiders.MatchLobbyApplication`, when a
  conforming implementation is checked with
  `contract4chessraiderstest.CheckMatchLobbyApplication`, then it must pass
  the create → join → ready → start lifecycle sequencing, including
  rejecting a premature `Start` with `ErrTeamsNotViable`/`ErrNotAllReady` and
  rejecting a non-creator `Start` with `ErrNotCreator`.
- AC-3: Given the package's exported identifiers, when counted as
  types+interfaces+error-vars (excluding individual enum constant values),
  then the total is within the founder-approved ballpark of "roughly 30"
  (actual: 14 types/interfaces + 15 errors = 29), and every inclusion traces
  to either an existing decision (0010–0014) or an existing, currently
  ad-hoc-duplicated integration seam — never padding to hit the number.
- AC-4: Given `sneat-go`'s `pkg/modules/chess` package, when
  `provideChessInternal()` is called, then it returns exactly four
  constructed values satisfying `MatchLobbyApplication`,
  `IdentityLinkApplication`, `PortalSessionApplication`, and
  `contract4competios.GameLauncher` respectively, with no gameplay or
  rules-evaluation logic added to `sneat-go` in the process (wire-and-
  configure only, per `sneat-go/AGENTS.md`).

## Open Questions

- **OQ-1: No existing chessraiders decision or Feature spec scopes a
  platform-wiring contract package by name.** A dedicated research pass
  across `spec/decisions/0001`–`0017` and every `spec/features/**` README in
  `sneat-co/chessraiders` (as of `origin/main`) found no mention of
  `ext-chessraiders`, `contract4chessraiders`, or a "public API"/SDK for
  platform wiring. The public-surface work that *does* exist there
  (`publish-the-standard-bot`) is a different initiative for a different
  audience (Starlark bot authors, landing in `sneat-games/chessraiders/go`
  as real runtime code, not a wiring contract). This Feature's scope — lobby
  vocabulary, external identity, portal session binding — was derived from
  (a) decision 0012's literal prose shape, (b) decisions 0010/0011/0013's
  portal/identity framing, and (c) the real, currently ad-hoc-duplicated
  DTOs between `sneat-go` and the private `sneat-bots` repo. It was not
  handed down by an existing chessraiders decision. **Ask:** should
  `sneat-co/chessraiders` record a companion decision (0018?) that formally
  scopes this contract's boundary, so future changes to the lobby/identity/
  portal seams have a recorded place to check against instead of only this
  Feature living in a different repo?
- **OQ-2: `provideChessInternal()` is a new naming convention, not an
  existing one.** A full-tree search of `sneat-go` found zero existing
  `provide<X>Internal` functions for any extension (competios, gameboard,
  eventius included) — this Feature establishes the pattern rather than
  mirroring precedent. **Ask:** should this naming convention be written
  into `sneat-go/AGENTS.md` as the platform's standard shape for a `*-contract`
  wiring entry point, so the next extension has a real precedent instead of
  reading this one Feature?
- **OQ-3: Conformance-checker coverage is deliberately partial.** Only
  `MatchLobbyApplication` has a shared conformance harness in
  `contract4chessraiderstest`; `IdentityLinkApplication` and
  `PortalSessionApplication` do not. **Ask:** is that an acceptable first
  release, or does the founder want conformance coverage for all four
  interfaces before this Feature can move past `Implementing`?

---
*This document follows the https://specscore.md/feature-specification*
