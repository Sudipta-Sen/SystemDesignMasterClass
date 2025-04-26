# Flash Sale System Design

## Objective

Design a system that can handle high-volume, high-speed purchases during flash sales like Amazon's or Flipkart's Big Billion Days, ensuring fairness, minimal contention, and system performance.

## Initial Approach: Shared Row for Product

- Design: One row per product with a quantity field.
- Transaction Flow:

    ```sql
    BEGIN;
    SELECT * FROM items WHERE id=? FOR UPDATE;
    UPDATE items SET quantity = quantity - 1 WHERE id=?;
    INSERT INTO cart (...);
    COMMIT;
    ```
- Problem:
    - Only one transaction can access the row at a time.
    - Causes high contention, long waits, and slows the system.

## Optimized Approach: One Row per Unit

- **Design:** Each unit of a product is a separate row in the items table.
    - 10 units of a product = 10 rows.

- **Why This Works:**
    - Reduces lock contention.
    - Enables multiple users to grab different rows simultaneously.

- **Improved SQL with SKIP LOCKED:**
    ```sql
    BEGIN;
    SELECT * FROM items
    WHERE product_id=? AND picked_at IS NULL
    ORDER BY id
    LIMIT 1
    FOR UPDATE SKIP LOCKED;

    UPDATE items
    SET picked_at = NOW(), picked_by = <user_id>
    WHERE id = ?;

    INSERT INTO cart (...);
    COMMIT;
    ```

- Key Concepts:

    - `FOR UPDATE SKIP LOCKED:` Skip already locked rows, ensuring non-blocking concurrent access.

    - `picked_at:` Marks the time item is added to the cart.

## Payment Phase

- Happens separately after adding to cart.
- If payment succeeds → mark `purchased_at`.
- If payment fails → `picked_at` is cleared via a cron job, making the item available again.

    ```sql
    UPDATE items
    SET picked_at = NULL
    WHERE payment_failed AND picked_at IS NOT NULL;
    ```

##  Notification Strategies for Re-availability

1. **No Notification:** Only users revisiting during the sale get access.
2. **Under-sell Strategy:**

    - Don't re-list items after failed payments.
    - Better for user experience, avoids false hope.

3. **Over-sell Strategy:**
    - Like IRCTC — allow a waitlist but comes with its own complexities.

## Real-World Analogies
- Mall Shopping:
    - Pick items into your cart first → pay at counter → failure = item returned to shelf.

- BookMyShow:
    - Undersell model.
    - Delay between payment failure and seat becoming available again.

- IRCTC:
    - Oversell model — uses waitlists.