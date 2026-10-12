# Box-for-Credit Clothing Swap — Build Spec

Source: *Box-for-Credit Clothing Swap — PRD* (Bridget Brooksby, Oct 12, 2026, ENT 401 prototype).
This spec turns the PRD's decisions into what to build in Glide, Google Sheets and Stripe. Anything the PRD leaves open is marked **TBD** and listed in [Open items](#open-items); nothing here invents a number the PRD doesn't give.

## 1. Scope

**Build:** a Glide app for BYU women where members drop off clothes for store credit and anyone can shop the resulting inventory, with campus drop-off and pickup slots.

**Do not build:** men's clothing, non-BYU users, mail-in boxes, shipping, home pickup, a customer-facing warehouse store, peer-to-peer listing, or automated pricing.

## 2. Stack

| Piece | Tool | Role |
| --- | --- | --- |
| App (UI, login, logic) | Glide | All user and admin screens |
| Database | Google Sheet behind Glide | Tabs listed in §4 |
| Payments | Stripe | $6.99/month membership, and the cash part of purchases |
| Photos | Glide image upload | One photo per dropped-off item |

"Cash" in this spec means a card payment through Stripe, as opposed to store credit.

## 3. Roles

| Role | How assigned | Can |
| --- | --- | --- |
| Non-member | Any signed-in user without an active membership | Browse the store, buy with cash |
| Member | Active $6.99/month Stripe subscription | Everything a non-member can, plus submit drop-offs, hold a credit balance, pay with credit, see member prices |
| Admin | Listed by email in the Users tab (`Role = Admin`) | Grade items, manage tiers and slots, mark pickups complete |

If a membership lapses, the user becomes a non-member: their credit balance stays but can't be spent or added to until they renew (see [Open items](#open-items) to confirm).

## 4. Data model (Google Sheet)

### Users
| Column | Type | Notes |
| --- | --- | --- |
| UserID | text | Glide row ID |
| Email | email | Login identity |
| Name | text | |
| Role | choice | `Member`, `NonMember`, `Admin` |
| MembershipStatus | choice | `Active`, `Lapsed`, `None`; updated from Stripe |
| StripeCustomerID | text | |
| CreditBalance | number (computed) | Sum of this user's rows in Credit |

### Tiers
Maintained by hand.
| Column | Type | Notes |
| --- | --- | --- |
| Tier | text | e.g. Tier 1, Tier 2 — **TBD** |
| Brands | text | Brands accepted in this tier — **TBD** |
| ListPrice | currency | Store price for items in this tier — **TBD** |
| Credit | currency (computed) | `ListPrice × 0.60` |

### DropOffs
| Column | Type | Notes |
| --- | --- | --- |
| DropOffID | text | |
| UserID | relation → Users | Member who dropped off |
| SlotID | relation → Slots | Campus drop-off slot |
| SubmittedAt | datetime | |
| Status | choice | `Submitted`, `Graded` (all items graded) |

### Items
| Column | Type | Notes |
| --- | --- | --- |
| ItemID | text | |
| DropOffID | relation → DropOffs | |
| OwnerUserID | relation → Users | The member who dropped it off |
| Photo | image | Uploaded by the member |
| Brand | text | Entered by the member, confirmed at grading |
| Size | choice | Filter field |
| Category | choice | Filter field (tops, dresses, etc. — list **TBD**) |
| EstimatedTier | relation → Tiers | Looked up from the member's brand entry |
| EstimatedCredit | currency (computed) | `EstimatedTier.Credit`; blank if brand isn't in any tier |
| Status | choice | See §5 |
| GradedTier | relation → Tiers | Set by admin on accept |
| ListPrice | currency (computed) | `GradedTier.ListPrice` |
| CreditAwarded | currency | `ListPrice × 0.60`, written once at grading |
| ListedAt | date | Set at grading on accept |
| DaysListed | number (computed) | `Today − ListedAt` |
| MemberPrice | currency (computed) | `ListPrice × 0.70` if `DaysListed ≥ 30`, else `ListPrice` |
| GradedBy / GradedAt | text / datetime | Audit |
| RejectReason | text | Optional, shown to the member |

### Credit (ledger)
Balances are never edited directly; every change is a row. A user's balance is the sum of their `Amount` column.
| Column | Type | Notes |
| --- | --- | --- |
| EntryID | text | |
| UserID | relation → Users | |
| Amount | currency | Positive = earned, negative = spent |
| Type | choice | `Earned`, `Spent`, `Adjustment` |
| ItemID / OrderID | relation | What caused it |
| CreatedAt | datetime | |

### Orders
| Column | Type | Notes |
| --- | --- | --- |
| OrderID | text | |
| BuyerUserID | relation → Users | |
| ItemID | relation → Items | One item per order for the prototype |
| PriceCharged | currency | `MemberPrice` for members, `ListPrice` for non-members |
| CreditUsed | currency | 0 for non-members |
| CashPaid | currency | `PriceCharged − CreditUsed` |
| StripePaymentID | text | Blank if `CashPaid = 0` |
| PickupSlotID | relation → Slots | |
| Status | choice | `Paid`, `PickedUp`, `Cancelled` |
| CreatedAt | datetime | |

### Slots
| Column | Type | Notes |
| --- | --- | --- |
| SlotID | text | |
| Type | choice | `DropOff`, `Pickup` |
| Location | text | Campus location — **TBD** |
| Start / End | datetime | |
| Capacity | number | Optional |

## 5. Item lifecycle

```
Submitted ──accept──▶ Listed ──purchase──▶ Sold ──pickup──▶ PickedUp
    │
    └──reject──▶ Rejected (donated)
```

- **Submitted:** member uploaded a photo; item is physically in the drop-off bag.
- **Listed:** admin accepted it and set a tier. Visible in the store.
- **Sold:** an order was paid. Hidden from the store immediately.
- **PickedUp:** admin confirmed handoff at the pickup slot.
- **Rejected:** admin rejected it; it is donated, and the member sees it marked rejected.

## 6. Flows

### 6.1 Sign up and membership
1. User signs in to Glide with email.
2. A new user is a `NonMember` and can browse and buy immediately.
3. "Become a member" starts a $6.99/month Stripe subscription. On success, `MembershipStatus = Active`, `Role = Member`.
4. When Stripe reports a failed renewal or cancellation, set `MembershipStatus = Lapsed`, `Role = NonMember`.

### 6.2 Drop-off (member only)
1. Member picks a campus drop-off slot.
2. For each item: upload a photo, enter brand, size and category.
3. The app shows an estimated credit per item and a total, looked up from the brand's tier. Brands not in any tier show "not likely accepted".
4. Member submits; a DropOff row and one Items row per item are created with `Status = Submitted`.
5. Member brings the bag to the slot.

### 6.3 Grading (admin)
1. Admin opens the grading queue: all `Submitted` items, grouped by drop-off, oldest first, with photos.
2. For each item, admin chooses:
   - **Accept:** pick the tier (defaults to `EstimatedTier`). The app sets `Status = Listed`, `ListedAt = today`, `CreditAwarded = ListPrice × 0.60`, and adds a `Credit` row `+CreditAwarded` for the owner.
   - **Reject:** optional reason. The app sets `Status = Rejected`. No credit.
3. When every item in a drop-off is graded, set the DropOff `Status = Graded`. The member's drop-off view shows each item as accepted (with credit) or rejected.

Target: credit lands within a few days of drop-off.

### 6.4 Shopping (anyone)
1. Store shows all `Listed` items. Filters: size, brand, category.
2. Price shown: members see `MemberPrice` (with the original struck through when discounted); non-members see `ListPrice`.
3. Checkout:
   - **Member:** credit applies first, up to the lesser of balance and price. The member can choose to use less credit. Any remainder is paid by card through Stripe.
   - **Non-member:** full price by card through Stripe.
4. Shopper picks a pickup slot.
5. On successful payment (or immediately if fully paid with credit): create the Orders row with `Status = Paid`, add a `Credit` row `−CreditUsed` if any, and set the item `Status = Sold`.
6. At the pickup slot, admin marks the order `PickedUp`.

### 6.5 Aging price drop
`MemberPrice` is a computed column, so the 30% member discount applies automatically once `DaysListed ≥ 30`. Non-member price never changes. Credit already awarded to the original owner is not changed by the drop.

## 7. Business rules

1. Credit for an accepted item = 60% of its tier's list price, set once at grading.
2. Members get 30% off items listed 30 days or more; non-members pay list price.
3. Women's clothing only; only brands in the Tiers tab are accepted.
4. Rejected items are donated, and the member is shown which items were rejected.
5. Credit can only be spent in the store and is never cashed out.
6. Only members can drop off or spend credit.
7. A credit balance can never go below zero.
8. An item can be sold once. If two shoppers check out the same item at once, the second order is refunded and cancelled by an admin (acceptable at prototype volume).

## 8. Screens

**Shopper (everyone):** Store (grid + filters), Item detail, Checkout, My orders (with pickup slot).
**Member only:** Credit balance + ledger history, New drop-off, My drop-offs (status and per-item result), Membership.
**Admin only:** Grading queue, Tiers editor, Slots editor, Pickups (today's orders to hand off), Metrics.

## 9. Metrics (admin view)

Computed from the Sheet and checked weekly during the pilot.

| Metric | Target | Source |
| --- | --- | --- |
| Survey responses | 100 | Survey tool (outside the app) |
| Paid members | 25 | Users where `MembershipStatus = Active` |
| Drop-offs | 40 | DropOffs rows |
| Items listed | 200 | Items ever set to `Listed` |
| Items sold | 75 | Orders where `Status ≠ Cancelled` |
| Members who both drop off and shop | 50% | Members with ≥1 drop-off and ≥1 order ÷ members with ≥1 drop-off |

## 10. Acceptance tests

Stage 2 of the PRD's plan: run five test items through the whole app before the pilot. It passes when every check below holds and every Credit balance matches a hand calculation.

1. A new user can browse and buy with cash but can't open New drop-off.
2. After paying $6.99, the user is a member and can submit a drop-off.
3. Each submitted item shows an estimated credit matching its brand's tier.
4. Accepting an item lists it at the tier price and adds exactly 60% of that price to the owner's balance.
5. Rejecting an item adds no credit and shows it as rejected to the owner.
6. A member checkout using credit only reduces the balance by the price and charges nothing in Stripe.
7. A member checkout using credit and cash reduces the balance by the credit used and charges the rest in Stripe.
8. A non-member checkout charges the full list price in Stripe.
9. A sold item disappears from the store.
10. An item with `ListedAt` set 30 days back shows 70% of list price to members and full price to non-members.
11. Marking an order picked up sets it `PickedUp`.
12. A member's balance equals the sum of their Credit rows, and can't go negative.

## Open items

These come from the PRD's open questions and block or shape parts of the build.

| Item | Blocks |
| --- | --- |
| Brands per tier and each tier's list price | Tiers tab, estimated credit, grading |
| Category list | Store filters, drop-off form |
| Campus drop-off and pickup locations/times, and BYU permission | Slots tab, launch |
| Whether credit can ever be cashed out (spec assumes never, per the business rules) | Rule 5 |
| What happens to credit when a membership lapses (spec assumes kept but frozen) | §3 |
| Whether Glide is acceptable to the professor | Whole stack |
| Storage unit lease allows business use | Grading location |
| Sources for the 11M and 20,000 market figures | Not a build blocker |
