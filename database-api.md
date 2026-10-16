# Database API

The cashier and manager screens use these JDBC repositories. Construct them with
`Database`, call them from the shared background executor, and close returned
connections. Repository methods close their own connections and result sets.
Configuration failures throw `IOException`; database failures throw `SQLException`;
invalid inputs throw `IllegalArgumentException`.

## Checkout

`OrderRepository.submit(OrderRequest, Map<Integer, BigDecimal> quotedPrices)`
returns `OrderReceipt` only after committing. The overload without a price map
uses current menu prices. Nested records are:

- `OrderRequest(customerId, employeeId, paymentMethod, taxAmount, tipAmount, lines)`
- `OrderLine(menuItemId, quantity, iceLevel, sugarLevel, toppings)`
- `Topping(inventoryId, servings, unitPrice)` — servings per drink; price is the captured quote.
- `OrderReceipt(orderId, subtotal, taxAmount, tipAmount, totalAmount)`

Checkout validates an active employee, available/in-season drinks, recipes, and
current topping availability/prices. Stale quotes reject the entire order before
inserts. All consumed inventory rows are locked in ascending ID order; recipe
and extra-topping quantities are aggregated before deduction. Orders, lines,
toppings, and stock changes commit together or roll back together.

`CheckoutPolicy` supplies the 8.25% tax rate and allowed ice/sugar choices to the
UI and validation. `CheckoutCart` captures immutable selections and rounds tax
once per order; the repository receives exact-cent tax/tip amounts. Purchase
prices are saved on order lines so later catalog price edits do not rewrite sales.

The transaction uses repeatable-read isolation. Conflicting writes may raise
SQLSTATE `40001`; domain rejections use `P0001`. The GUI retains failed carts and
instructs the cashier to refresh/rebuild stale selections. If connectivity fails
during commit, reconcile order history before retrying: there is no idempotency
key. Selecting Card records the payment method; this app does not charge cards.

## Inventory and employees

`InventoryItem` contains ID, ingredient name, unit, quantity, target fill level, minimum stock,
`topping`, and `toppingPrice`. Quantities allow three decimal places; prices allow
two. Minimum stock must be between zero and the target; zero disables low-stock alerts.
A topping requires a nonnegative price; a non-topping has a null topping price.

- `listAll()` loads inventory, including toppings.
- `add(name, unit, quantity, target, minimum)` creates ordinary stock; the overload adds
  `boolean topping, BigDecimal toppingPrice`.
- `adjustQuantity(id, delta)` atomically adds stock; negative results are rejected.
- `setQuantity(id, expectedQuantity, newQuantity)` rejects stale absolute edits.
- `update(id, name, expectedQuantity, newQuantity, target, minimum, topping, toppingPrice)`
  edits details while preserving units already used by recipes.
- `restockLowStock()` atomically fills items below their saved minimum to their current
  targets and returns the number updated. This is the restock report's threshold.

`EmployeeRepository` provides `listAll`, `add(name, role)`, `update(id, name, role)`,
and `setActive(id, active)`. Employees with sales history are deactivated rather
than deleted. `Database.getMenuItems()` loads available drinks and seasonal dates;
`getToppings()` loads designated toppings with positive stock.

## Menu and recipes

`MenuRepository.listAll()` includes unavailable items and recipes.
`add(MenuDraft)` and `update(id, MenuDraft)` save menu metadata and its complete
recipe in one transaction. `MenuDraft` contains name, description, price,
availability, optional inclusive seasonal start/end dates, and a nonempty list
of `RecipePart(inventoryId, servings)`. Both dates must be blank or form an ordered
range. Servings use each inventory item's unit; include supplies such as cups.

Changes retain the menu ID and historical purchase prices. Invalid ingredients
roll back the whole save. `delete(id)` removes an unsold item and its recipe;
items with sales history must instead be marked unavailable.

## Reports

All queries are packaged from `backend/queries/`, use prepared date parameters
when needed, run in read-only transactions, and have a 15-second query timeout.
Only trusted packaged SQL is accepted. The simple loader supports line comments
and semicolon-separated statements, not semicolons inside SQL string literals.

- `run(Report)` returns a detached `ReportTable(columns, rows)` with JDBC values,
  including SQL nulls. The enum has 16 choices: ten normal queries, five special
  queries, and product usage. Each retains its SQL's documented status/date filters.
- `run(PRODUCT_USAGE, startDate, endDate)` requires an inclusive ordered window.
  It returns inventory ID/name/unit, recipe usage, extra-topping usage, and total.
  Recipe usage uses **current recipes**, so recipe edits can change historical
  usage; extras use saved topping servings. Different units are never summed together.
- `salesByItem(from, to)` returns `SalesReport` with per-item units, drink revenue,
  topping revenue, and combined revenue for completed orders. Both dates are
  included; tax/tips are excluded. Toppings are aggregated before joining to avoid
  multiplying drink quantities. Items without sales in the window are omitted.
- `runXReport()` defaults to today; `runXReport(date)` accepts another business date.
  `XReportSummary` includes completed-order counts, subtotal, tax, tips, total,
  and 24 `XReportHour` rows, including empty hours. It never closes or resets sales.

Date windows compare the stored PostgreSQL `TIMESTAMP` values. Reports are separate
from checkout and never modify order history. Results are not paginated.

## Verification

From `frontend/`, `mvn clean verify` runs offline tests. Set `POS_DB_TEST=true`
to additionally check the configured class database without writing to it.
For repository write/rollback/concurrency tests, use a disposable local database:

```sh
POS_REPOSITORY_TEST=true \
REPOSITORY_TEST_URL='jdbc:postgresql://localhost:5432/postgres?user=postgres' \
mvn clean verify
```

Supply the local test server's credentials in the URL as needed. Each test creates
and drops its own schema; never point these tests at the shared class database.
Rolled-back inserts can leave identity-sequence gaps, which is normal PostgreSQL behavior.
