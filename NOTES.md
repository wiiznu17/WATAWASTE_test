## BE-1 — Exception on checkout
**What I checked:**
- Called checkout (`POST /v1/th/orders`) with items in cart -> received 500 error.
- Server traceback pointed to `order.py:69` (`create_from_cart`) throwing `KeyError: 'mealId'`.

**Root cause:**
- `lines[0].model_dump()` returns `snake_case` keys (`meal_id`), but the code accessed `first_line["mealId"]`.
- The entire block was also unused dead code (fetched `meal.store_id` into throwaway variable `_`).

**Fix:**
- Commented out the unused dead code block in `order.py`.
- Checkout now succeeds (201), empties the cart, and retains reserved stock.

## BE-2 — Logical pricing bug
**What I checked:**
- Checked `GET /v1/th/cart` after adding items. Subtotal was charged at full original price instead of discounted price.

**Root cause:**
- In `cart.py:38` (`get_cart`), `unit_price` was assigned from `meal.original_price` instead of `meal.discounted_price`.

**Fix:**
- Changed to `unit_price = meal.discounted_price`.
- Cart `unit_price`, `line_total`, and `subtotal` now correctly reflect discounted pricing.

## BE-3 — Logical inventory bug on cancel
**What I checked:**
- Placed an order with quantity 3 and cancelled it. Stock only restored by 1 unit instead of 3.

**Root cause:**
- In `order.py:cancel()`, the inventory restoration loop called `stock.apply()` with a hardcoded `quantity=1`.

**Fix:**
- Updated to `quantity=line.quantity`.
- Cancelling an order now properly restores the exact ordered quantity.

## BE-4 — Logical inventory bug on cart quantity increase
**What I checked:**
- Increased cart item quantity via `PATCH /v1/th/cart/items/{id}`. Store stock increased instead of reserving additional units.

**Root cause:**
- In `cart.py:update_item()`, the `delta > 0` condition used `event_type=StockEventType.INCREMENT` instead of `DECREMENT`.

**Fix:**
- Changed event type to `StockEventType.DECREMENT`.
- Increasing cart quantity now correctly decrements store stock.

## FE-1 — Wrong "You pay" price
**What I checked:**
- On the Meals page, meal cards showed the original price under "You pay" instead of the discounted price.

**Root cause:**
- `MealsPage.tsx:66` rendered `meal.original_price` inside `<strong className="pay">`.

**Fix:**
- Changed to `meal.discounted_price`.
- Meals page now displays original price struck-through and discounted price in bold.

## FE-2 — Stock events filter
**What I checked:**
- Filtering by `meal_id` on `/stock-events` returned all events without filtering.

**Root cause:**
- `FE/src/api/client.ts:listStockEvents` sent query parameter `?meal=`, but backend endpoint expects `?meal_id=`.

**Fix:**
- Updated query parameter to `?meal_id=`.
- Stock events page now properly filters events by meal ID.