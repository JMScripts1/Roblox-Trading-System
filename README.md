# Trading System

A server-authoritative player-to-player trading system for Roblox, written in strictly typed Luau. It doesn't own any player data: it talks to your inventory and currency code through two small adapter files, so it drops into an existing game without rewriting either side.

Works out of the box with [Roblox-Inventory-System](https://github.com/JMScripts1/Roblox-Inventory-System).

> **Status:** in active development. The sections below describe the finished system; see [Roadmap](#roadmap) for what is complete.

<!-- showcase video / gif goes here -->

## Features

- Send, accept, and decline trade requests, with a cooldown and timeout
- Both players build an offer of items and currency
- Any change to either offer clears both players' ready state, so nobody gets swapped out at the last second
- A short confirm countdown after both players are ready
- Everything is re-checked at the moment of the swap, and the swap is all-or-nothing
- Both players' data is saved right after a trade, shrinking the window for rollback dupes

## Integrating it into your game

Trading never reads or writes player data itself. The server side goes through `TradeAdapter`:

```lua
TradeAdapter.exists(itemId)                      -- the item is defined in your game
TradeAdapter.isTradable(itemId)                  -- false for items that should never change hands
TradeAdapter.has(player, offer)                  -- owns every item and currency amount in the offer
TradeAdapter.fits(player, incoming, outgoing)    -- room for everything incoming once outgoing is gone
TradeAdapter.apply(player, incoming, outgoing)   -- swap one side; only called after both players pass
TradeAdapter.save(player)                        -- force a save after the trade
```

`apply` must not yield, and must not fail once `has` and `fits` have passed. That's what keeps the swap all-or-nothing.

The client side goes through `ItemInfo`, which gives the UI an item's name, icon, and rarity.

The default adapters target Roblox-Inventory-System. To use your own inventory, rewrite those two files against it; nothing else changes.

| Module | What it controls |
|---|---|
| `TradeAdapter` | How the server talks to your inventory and currency |
| `ItemInfo` | How the UI looks up item names, icons, and rarities |
| `Config` | Offer size, cooldowns, countdown length, tradable currencies |
| `Theme` | UI colors, fonts, and corner radius |

## Architecture

The client never decides game state. It sends an intent ("offer 3 of this item", "I'm ready"), and the server validates and applies it.

Every request goes through the same checks, in order:

1. Rate limit per player, per action
2. Type check every argument
3. Range and existence check (item exists, no NaN or infinity, counts are whole numbers)
4. Permission check (player is in this trade, owns what they offered)
5. Apply
6. Send both players only what changed

A trade is a small state machine on the server:

```
Requested -> Open -> BothReady -> Countdown -> Complete
                ^          |          |
                +----------+----------+   any offer change resets to Open
```

Leaving, a timeout, or a failed final check cancels the trade, and nothing moves.

```
src/
  shared/      config, theme, types
  server/      TradeService, TradeAdapter
  client/      TradeController, ItemInfo, UI components
```

## Running locally

Requires [Rokit](https://github.com/rojo-rbx/rokit) and the [Rojo](https://rojo.space) Studio plugin.

```bash
rokit install
rojo serve
```

## Tooling

- **Rojo** for syncing files into Studio
- **luau-lsp** for type checking, with `--!strict` in every module
- **StyLua** for formatting
- **Selene** for linting

## Roadmap

- [ ] Adapter and the inventory-side additions it needs
- [ ] Trade sessions: request, accept, decline, cancel, timeout
- [ ] Offers, ready state, countdown, and the final swap
- [ ] Placeholder UI
- [ ] Edge cases: leaving mid-trade, items used mid-trade, full inventory, request spam, malformed requests
- [ ] Final UI with themed styling, transitions, and tooltips
