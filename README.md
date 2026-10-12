# ReRack: clothing swap prototype

A clickable prototype of the Box-for-Credit clothing swap described in [SPEC.md](SPEC.md). Open `index.html` in any browser. There's no install step and no server. Everything is saved in that browser's local storage.

The real app is planned in Glide on a Google Sheet, with Stripe for payments (see SPEC.md). This prototype mirrors its flows and rules so they can be tried before the Glide build.

## How it works

- **Accounts**: anyone with a BYU email (`@byu.edu`) can create a free account and shop at non-member prices, paying by card.
- **Membership**: $6.99/month. Members can drop off clothes, earn and spend store credit, and get member prices on older items. If a membership is cancelled, the credit is kept but frozen until the member renews.
- **Drop off**: members pick a campus drop-off slot, then add each piece with a photo, brand, category and size. The app shows an estimated credit from the brand's tier. Brands that aren't on any tier are flagged "not likely accepted".
- **Grading**: in Admin mode, each piece is accepted with a tier or rejected. Accepted pieces are listed in the store right away at the tier's price, and the member gets credit worth 60% of that price. Rejected pieces are donated, and the member sees which ones and why.
- **Shopping**: anyone can browse and filter by category, size and brand. Members pay with credit first (they can choose to use less), and any remainder goes on their card. Each order is one item, picked up at a campus pickup slot.
- **Price drop**: items listed 30 days or more are 30% off for members. Non-members always pay the list price, and credit already given for an item doesn't change.
- **Store credit**: credit is a ledger of earned and spent entries, and the balance is their sum. It can't go negative and can't be cashed out.
- **Admin mode**: turn it on under Account to see the Admin tab, where you can grade drop-offs, mark pickups done, edit tiers and slots, see the pilot metrics, and move a listing back 30 days to test the price drop.

## Tuning

Everything that drives the business is in the `CONFIG` block at the top of the script in `index.html`:

- `MEMBERSHIP_PRICE`, `CREDIT_RATE` (0.60), `AGED_DAYS` (30), `AGED_DISCOUNT` (0.30)
- `DEFAULT_TIERS`: brands and list price for each tier
- `CATS` and `SIZES`: the category and size options
- `SLOT_PLAN`: default campus locations and times for drop-off and pickup slots
- `REJECT_REASONS`: the reasons graders can show members

**Placeholders**: the tier brands and prices, the category list, and the campus slot locations and times are made up. SPEC.md lists them as open items.

## What a production version still needs

See SPEC.md for the full build plan. Compared with the real app, this prototype:

- Keeps one account per browser, so other members, admin grading and the metrics all run on this one device's data.
- Uses a toggle for admin access instead of an admin role on the account.
- Takes card details without charging anything; the real app uses Stripe for the membership and for card payments.
- Stores shrunk photos in the browser instead of uploading them.
