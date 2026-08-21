# Custodia — Technical Whitepaper

**On-chain resale price enforcement for event ticketing, deployed on GIWA Sepolia.**

---

## 1. Abstract

Ticket scalping persists because resale-price enforcement has historically lived inside centralized platform policy rather than in the transaction layer itself. Custodia removes that gap: the maximum allowable resale price for any ticket is computed and enforced by the smart contract at the moment of transfer, making unauthorized markups a reverted transaction rather than a policy violation. This document describes the system architecture the on-chain mechanisms that make the anti-scalping guarantee hold, and the technical tradeoffs made in the current implementation.

---

## 2. System Overview

Custodia is a single-contract protocol (`Custodia.sol`, Solidity `^0.8.20`) deployed on GIWA Sepolia. It manages four core resources:

- **Events** — created and owned by an organizer address
- **Tiers** — nested pricing structures within an event
- **Tickets** — individually tracked ownership records (not a standard ERC-721, described in §6)
- **Waitlist Entries** — per-event queues activated once a tier sells out

All state transitions — purchase, resale, check-in, cancellation, refund, and waitlist promotion  are handled by dedicated external functions with role-scoped access control (`onlyOwner`, `onlyOrganizer`, `eventExists` modifiers)

---

## 3. Core Mechanism: On-Chain Resale Cap

This is the architectural centerpiece of the protocol

### 3.1 Cap Definition
Each event stores a `resaleCapBps` value (basis points, denominator `10000`) at creation time. A cap of `11000` means a ticket may never resell for more than 110% of its current holder's purchase price

### 3.2 Why Purchase Price, Not Face Value
The cap is computed against `ticket.purchasePrice`  the amount the *current* holder actually paid — not the original tier price. This is deliberate:

```solidity
uint256 maxAllowed = (t.purchasePrice * e.resaleCapBps) / BPS_DENOMINATOR;
require(msg.value <= maxAllowed, "Exceeds resale price cap");
```

If the cap were pinned to original face value, a ticket could theoretically compound markups across repeated resales without ever tripping a check relative to its most recent sale. Anchoring to current purchase price means every single resale  no matter how many times a ticket changes hands  is capped relative to what the last person actually paid, closing that loophole.

### 3.3 Enforcement Point
The check occurs inside `resellTicket()`, before any funds move and before ownership updates. There is no separate "approval" or "listing" step that could be bypassed — the price check and the transfer are atomic within a single transaction.

---

## 4. Tiered Pricing & Dynamic Progression

Organizers are not constrained to a fixed tier taxonomy. `addTier()` accepts an arbitrary name, price, and supply, and tiers can be configured to auto-open:

```solidity
function _autoOpenNextTier(uint256 _eventId, uint256 _justClosedIndex) internal {
    uint256 nextIndex = _justClosedIndex + 1;
    if (nextIndex < events[_eventId].tierCount) {
        Tier storage nextTier = eventTiers[_eventId][nextIndex];
        if (nextTier.sold < nextTier.totalSupply) {
            nextTier.open = true;
            emit TierOpened(_eventId, nextIndex);
        }
    }
}
```

This is called automatically at the tail of `buyTicket()` whenever a tier's `sold` count reaches its `totalSupply`. Organizers retain manual override via `setTierOpen()` for cases where auto-progression isn't desired (e.g. holding a tier back for a scheduled release time)

---

## 5. Waitlist System — Dual Mode Design

When a tier is sold out, `joinWaitlist()` queues a buyer against a specific `(eventId, tierIndex)` pair. Slot distribution is handled by `offerWaitlistSlot()`, and its behavior branches on the event's `WaitlistMode` enum set at creation:

| Mode | Behavior |
|---|---|
| `AutomaticTransfer` (0) | Organizer's call to `offerWaitlistSlot()` immediately mints the ticket to the waitlisted address, with payment matching tier price executed as part of the same transaction that promotes the entry. |
| `TimeWindowOffer` (1) | The entry is marked `offered` with an `offerExpiresAt` timestamp (`block.timestamp + offerWindowSeconds`). The waitlisted user must then call `claimWaitlistOffer()` themselves before the window closes, paying the tier price directly. |

This split exists because different event types have different fairness requirements  a free community meetup might prefer automatic distribution to minimize friction, while a high-demand paid event benefits from giving each waitlisted person a fair, bounded window to act before the slot passes to the next person in line.

---

## 6. Ticket Representation

Tickets are represented as a `mapping(uint256 => Ticket)` keyed by an incrementing `ticketCounter`, rather than as a standard ERC-721 token. Each `Ticket` struct tracks `owner`, `purchasePrice`, `checkedIn`, and `refunded` state directly.

**Rationale:** ERC-721's standard `transferFrom` semantics have no native concept of a price constraint  enforcing a resale cap on top of ERC-721 would require either a wrapper contract intercepting marketplace transfers (which cannot compel external marketplaces to route through it) or a custom `_beforeTokenTransfer` hook, which still can't prevent transfers that occur outside Custodia's own `resellTicket()` function. By keeping ticket transfer logic entirely inside a purpose-built function rather than inheriting generic transfer semantics, the resale cap cannot be bypassed by moving a ticket through a generic NFT marketplace. The tradeoff is that tickets aren't automatically visible in wallets/marketplaces that only index ERC-721  a deliberate constraint given the project's priority (enforceability over interoperability).

---

## 7. Check-In: Two Verification Levels

```solidity
function isValidTicketHolder(uint256 _ticketId, address _holder) external view returns (bool)
function checkIn(uint256 _ticketId) external // onlyOrganizer, sets checkedIn = true
```

`isValidTicketHolder` is a free, gasless view function — any door-scanning client can call it to confirm a wallet currently holds a valid, unrefunded ticket without writing any state. `checkIn` is a state-changing, organizer-gated function that permanently marks a ticket as used, intended for events that need an auditable attendance record. Events can use either, both, or neither depending on operational needs.

---

## 8. Cancellation & Batch Refunds

`cancelEvent()` is organizer-gated and simply flips a `cancelled` flag — it does not move funds. Refunding is a separate, explicit action via `refundAllTickets()`:

```solidity
function refundAllTickets(uint256 _eventId) external payable eventExists(_eventId) onlyOrganizer(_eventId) {
    ...
    for (uint256 i = 0; i < ticketIds.length; i++) {
        Ticket storage t = tickets[ticketIds[i]];
        if (!t.refunded) {
            totalNeeded += t.purchasePrice;
        }
    }
    require(msg.value >= totalNeeded, "Insufficient refund funds sent");
    // second loop pays out and marks refunded
}
```

The organizer funds the exact amount needed for all outstanding tickets in a single `msg.value`, and the contract iterates once to compute the total and once to execute transfers — separating the "how much is owed" calculation from the "pay everyone" action to avoid partial-refund states if the organizer underfunds the call.

**Known scaling constraint:** this loop is O(n) in ticket count per event. For very large events (many thousands of tickets), this could approach block gas limits — a known tradeoff addressed on the roadmap.

---

## 9. Fee Model

Two independent, admin-controlled fee surfaces exist:

- **Resale fee** — `resaleFeeEnabled` (bool) + `resaleFeeBps` (hard-capped at `2000`, i.e. 20%), deducted from seller proceeds at time of resale and tracked in `platformFeesCollected` for later `onlyOwner` withdrawal.
- **No primary-sale platform fee** — `buyTicket()` transfers the full `msg.value` directly to the organizer with no cut taken. Platform sustainability is scoped entirely to the optional resale fee, keeping primary ticket sales frictionless for organizers and buyers.

---

## 10. Payment Layer

All value transfer uses native ETH via `payable` functions and `msg.value` — there is no ERC20 token integration in the current version. This reflects GIWA Sepolia's current state (no live native stablecoin at time of deployment). The `payable`/`msg.value` pattern and an ERC20 `approve`/`transferFrom` pattern are structurally different enough that stablecoin support is planned as a distinct contract version rather than a patch (see Roadmap, Phase 2, in the main project README).

---

## 11. Access Control Summary

| Function | Caller Restriction |
|---|---|
| `createEvent`, `addTier`, `setTierOpen`, `cancelEvent`, `refundAllTickets`, `offerWaitlistSlot`, `checkIn` | `onlyOrganizer` (event-scoped) |
| `setResaleFee`, `withdrawPlatformFees` | `onlyOwner` (contract owner) |
| `buyTicket`, `resellTicket`, `joinWaitlist`, `claimWaitlistOffer`, `isValidTicketHolder` | Public / any address |

No function permits an organizer to act on another organizer's event — every organizer-gated function is scoped to `events[_eventId].organizer == msg.sender`.

---

## 12. Deployment Reference

| | |
|---|---|
| Network | GIWA Sepolia |
| Chain ID | `91342` |
| Contract Address | `0x62bb24bF96b52783146591398e783E5CA30e892f` |
| Verified Source | [sepolia-explorer.giwa.io/address/...?tab=contract](https://sepolia-explorer.giwa.io/address/0x62bb24bF96b52783146591398e783E5CA30e892f?tab=contract) |
| Deployment Transaction | [sepolia-explorer.giwa.io/tx/0x99e59e08...](https://sepolia-explorer.giwa.io/tx/0x99e59e08e8661165281521191cf9caee91f21e411a6b9b4ebb8d2bdd6b9d53a5) |
| Compiler | Solidity `v0.8.26`, optimizer enabled, 200 runs |

---

## 13. Summary

Custodia's central design claim is narrow and verifiable: **a ticket cannot be resold above its organizer-defined cap, under any code path, because the cap check and the transfer are the same atomic operation.** Every other feature  tiering, waitlists, check-in modes, refunds — is built around preserving that guarantee while giving organizers enough flexibility to run real events without needing to trust a platform's internal policy enforcement.

---

*Built by Rahman — [GitHub](https://github.com/rahmansial477) · [X](https://x.com/rahmansial477)*
