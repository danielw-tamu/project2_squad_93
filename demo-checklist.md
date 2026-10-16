# Phase 4 demo checklist

Use the class AWS PostgreSQL database and vendor-related data. Each teammate
explains and demonstrates their feature. Have the Phase 0 sketches available to
briefly explain design changes.

## Start from the submitted release

Configure `frontend/db.properties` using the example, then from the repository root:

```sh
cd frontend
mvn clean verify
mvn javafx:run
```

Click **Check connection**. Existing databases need migrations 001–005; do not
rerun schema/seed loading on the class database. `backend/load.sql` replaces data.

## Demonstrate all five features

1. **Seasonal item — Manager → Inventory / Menu:** add a legitimate vendor ingredient
   with a meaningful unit, minimum stock, refill target, and a low initial quantity.
   Add a new seasonal drink with
   dates including today and a recipe using the ingredient and necessary supplies.
   Refresh the cashier and show the drink and its seasonal dates.
2. **Restock — Manager → Inventory → Restock Report:** show the new ingredient below
   its saved minimum stock level. Use **Bulk Restock All** or restock the selected ingredient,
   then refresh to show the new quantity and cleared report. Bulk refill affects
   every qualifying item, so coordinate this with the team.
3. **Checkout / X-report — Cashier, then Manager → Sales → View X-report:** complete
   a vendor order today and record its receipt ID. Generate today's X-report; show
   the appropriate hourly row and total. Run it again to show unchanged totals.
4. **Sales report — Manager → Sales Report:** select an inclusive date window and
   show units and revenue by drink. Include today's order or a known populated
   historical window. Empty windows should display no sales.
5. **Product usage — Manager → Sales → PRODUCT USAGE:** select the same window and
   show recipe usage, extra toppings, and total usage per inventory unit. Current
   recipes determine historical recipe usage; explain this if recipes were edited.

A blank X-report or restock report can be correct: today may have no sales, or
all stock may be above the threshold. Prepare valid data rather than changing the
report date to substitute for the required current-day X-report demonstration.

## Quick regression checks

- A manager price change, disabled topping/drink, or expired season rejects a stale
  cart without changing orders or stock. Refresh and rebuild the cart afterward.
- Insufficient stock retains the cart; restock and retry.
- Confirm a receipt in a separate SQL session, replacing the example ID:

```sql
SELECT order_id, placed_at, subtotal, tax_amount, tip_amount, total_amount
FROM orders WHERE order_id = 1;
```

## Submit

- Publish a Phase 4 GitHub release from the tested main commit; submit its release
  URL to the release assignment, not a branch or pull-request URL.
- Paste the compile/run commands above into the demo assignment.
- Ensure each member has four commits or an explanation and has demonstrated
  their frontend/backend contribution. Complete individual peer evaluation separately.
