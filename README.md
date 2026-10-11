# ReRack: clothing swap prototype

A clickable prototype of a membership clothing-swap app. Open `index.html` in any browser. There's no install step and no server. Everything is saved in that browser's local storage.

## How it works

- **Membership**: $6.99/month. Sign-up only accepts Orem and Provo ZIP codes (84601–84606, 84057–84059, 84097). That's the location barrier.
- **Trade in**: members photograph each piece, pick the category, brand type, size, and condition, and see a ballpark range. Then they send the box for an estimate.
- **Estimate**: warehouse staff set the trend level and confirm condition. They can decline pieces that aren't on-trend or are too worn. The member accepts the offer and picks **ship it** (prepaid label) or **drop off** (drop-off code).
- **Inspection**: staff check the box when it arrives and accept or decline each piece. Accepted pieces become store credit and get posted to the home warehouse's shop automatically.
- **Pricing**: shelf price = fair resale value. The trader gets that value minus a small markup: $2, plus $1 if the piece is trending, plus $1 if it's new or like new. Each product page shows what the trader was paid next to the shelf price.
- **Shopping**: credit is applied first, and anything over goes on the member's credit or debit card. Orders from the home warehouse ship to the member's address.
- **Traveling**: members can switch to another warehouse (Salt Lake, St. George) and spend credit there. Those orders are **pickup only**, with a pickup code.
- **Warehouse staff mode**: turn it on under Account to play the staff side (estimates, inspection, posting) in the prototype.

## Tuning

Everything that drives the business is in the `CONFIG` block at the top of the script in `index.html`:

- `MEMBERSHIP_PRICE`
- `LOCATIONS`: warehouses, addresses, hours, and the ZIPs that can sign up. The addresses are placeholders.
- `CATS`: base resale value per category
- `TIERS`, `CONDS`, `TRENDS`: price multipliers
- `valuate()`: the payout and markup formula

## What a production version still needs

- A backend and database for accounts, inventory, and trade-ins shared across users. Right now each browser has its own copy.
- Real payments and recurring billing, for example Stripe Billing for the membership and Stripe Checkout for card top-ups.
- Shipping labels from a carrier API such as Shippo or EasyPost.
- A separate staff login instead of the demo toggle.
- Photo storage in the cloud. Right now photos are shrunk and kept in the browser.
