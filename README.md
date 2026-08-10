# ext-chessraiders

The public contract for **Chess Raiders** — the Sneat platform's real-time
multiplayer chess game. Types, interfaces, and errors only: what a host, a
Telegram/social bot shell, or a distribution-portal adapter needs in order to
integrate with Chess Raiders, without depending on the private engine.

The engine itself is private (`sneat-co/chessraiders`). This repository is
the seam a host or portal integrates against — it carries no board state,
move, or rules-evaluation logic, and imports nothing from
`server-go/chess` or any other private chessraiders package.

## What's here

| Package | Purpose |
|---|---|
| [`backend/contract4chessraiders`](backend/contract4chessraiders) | The contract: `Side`, `CaptureMode`, `Lifecycle`, `Player`, `LobbyView`, the `MatchLobbyApplication` / `ExternalIdentityVerifier` / `IdentityLinkApplication` / `PortalSessionApplication` interfaces, and their errors. |
| [`backend/contract4chessraiderstest`](backend/contract4chessraiderstest) | A conformance harness for `MatchLobbyApplication`. See "Conformance coverage" below for what it does not yet cover. |

```
go get github.com/sneat-co/ext-chessraiders/backend
```

## Why it is public, and why it has no dependencies

A contract that cannot be imported without credentials is not a contract — it
is a private type declaration that happens to be named like one. Every Sneat
extension is meant to import `*-contract` libraries freely and let the
application wire the implementation; that pattern only works if the contract
is reachable.

So this module holds **one invariant, enforced in CI: it depends on nothing
but the Go standard library.** No `require` block, no `go.sum`. A dependency
arriving here would reintroduce, one level down, exactly the credential
requirement this module exists to remove — so the build fails rather than
quietly acquiring one.

## Scope: what crosses this boundary, and what deliberately does not

Chess Raiders' private decision log (`spec/decisions/0010`–`0017` in
`sneat-co/chessraiders`) settles three integration seams that a host or
third party needs typed access to without private-repo credentials:

- **External identity** (decisions 0011–0013): a distribution portal
  (CrazyGames, itch.io) asserts a player's identity before Chess Raiders lets
  it act. `ExternalIdentity`, `ExternalIdentityVerifier`, and
  `IdentityLinkApplication` formalise the shape decision 0012 already
  describes in prose.
- **Portal session binding** (decisions 0010–0011): a distribution platform's
  own room/session concept (e.g. CrazyGames' `roomId`) binds to a match.
  `PortalSessionApplication` is that seam.
- **The lobby/social-shell boundary**: creating, joining, readying, and
  starting a match, and the presentation-neutral status view a Telegram or
  other bot shell renders. This formalises a vocabulary that already existed,
  duplicated ad hoc and without a shared contract, between the private
  `sneat-go` host and the private `sneat-bots` Telegram command package —
  `MatchLobbyApplication`, `LobbyView`, and their errors are the deduplicated,
  public version of that pre-existing shape.

**Explicitly excluded, with reasons:**

- **Gameplay** — moves, board state, legal targets, fog of war, morale,
  cooldowns, prisoners, fortifications, specialisation, espionage. All of it
  is derived from `server-go/chess` (private) and none of it crosses a public
  boundary; the private web/Mini App surface is the only place a game is
  played. Including any of it here would smuggle engine vocabulary into a
  contract one level removed from the engine itself.
- **The Competios `GameLauncher`/`ResultSink` seam** — Chess Raiders already
  implements `contract4competios.GameLauncher` directly
  (`facade4chess.NewCompetiosAdapter`, wired from `sneat-go`). Redeclaring it
  here would violate "reuse via stable facades," not honour it.
- **Match narration/event detail** (`TapEvent`'s rank, cargo count, capture
  outcome reason) — these are direct, one-level-removed projections of
  `server-go/chess` concepts (piece ranks, convoy cargo, capture resolution).
  A social/bot shell with private access already renders this; a public
  contract does not need to.
- **The Starlark bot-authoring surface** — a *different* public initiative
  (`publish-the-standard-bot`, landing in `sneat-games/chessraiders/go`)
  already publishes the bot runtime, tier parameters, and rules specs for bot
  authors. That is implementation code for a different audience (bot
  authors); this repository is a platform-wiring contract for hosts and
  portals. The two should not converge into one package.

## Conformance coverage

Only `MatchLobbyApplication` has a shared conformance checker
(`contract4chessraiderstest.CheckMatchLobbyApplication`). `IdentityLinkApplication`
and `PortalSessionApplication` were left without one in this first release —
their invariants (idempotent link, no silent rebind) are narrow enough for an
implementation's own unit tests to assert directly, and `MatchLobbyApplication`
has the only real lifecycle sequencing to get wrong. Adding checkers for the
other two is a reasonable fast-follow, not a gap silently carried forward.

## Relationship to `sneat-co/chessraiders`

The private engine's host composition (`sneat-go`) implements this contract's
interfaces via adapters and imports the private engine directly for
everything gameplay-related; this repository never imports the private
engine. Contract changes originate here and flow outward to implementors.
