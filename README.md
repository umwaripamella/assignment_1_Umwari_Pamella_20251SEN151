# PLSQL Assignment One – Sunrise Supermarket

**Name:** Umwari Pamella
**Student ID:** 20251SEN151
**Instructor:** Eric Maniraguha | **TA:** Afanyu Emmanuel
**Group:** D | **Submitted:** Wednesday, 23 September 2026
**Tool used:** MySQL, via phpMyAdmin (`localhost/phpmyadmin`)

---

## What this is

Sunrise Supermarket wants to understand its customers and its sales, so this
assignment builds a small four-table database for it — customers, products,
orders, and order_items — and then writes eight queries against it using
JOINs, a CTE, and window functions. Below is the schema, the sample data I
loaded, each query with an explanation of what it's actually answering, a
screenshot of it running in phpMyAdmin, and at the end a short read on what
the numbers say about the business.

## Database

- **customers** – who buys from the supermarket
- **products** – what's for sale, across 3 categories (Grains, Pantry, Dairy)
- **orders** – a purchase event: one customer, one date
- **order_items** – the line items inside an order (a product + a quantity)

Sample data: 6 customers, 8 products, 15 orders, 25 order items, spread
across January–April 2026 so a trend is actually visible. One customer,
Fabrice Rugamba, was deliberately left with zero orders so the LEFT JOIN in
Q3 has something to show.


## The 8 queries

### Q1 — Every order with the customer's name, city, and order date
*(INNER JOIN: orders + customers)*

This just answers "who ordered what, and when." Straightforward join on
`customer_id`.

```sql
SELECT o.order_id, c.customer_name, c.city, o.order_date
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
ORDER BY o.order_date;
```

![Q1 result](screenshots/q1_inner_join.png)

15 rows back — one per order, which is exactly what I'd expect since every
order in my data has a valid customer.

---

### Q2 — Every order item with product name, category, price, and quantity
*(JOIN: order_items + products)*

This breaks an order down into what was actually in it — useful for
figuring out which products move the most.

```sql
SELECT oi.order_item_id, oi.order_id, p.product_name, p.category, p.price, oi.quantity
FROM order_items oi
JOIN products p ON oi.product_id = p.product_id
ORDER BY oi.order_id, oi.order_item_id;
```

![Q2 result](screenshots/q2_join.png)

25 rows, matching the 25 order items I inserted.

---

### Q3 — All customers, with their orders where they exist
*(LEFT JOIN: customers + orders)*

An INNER JOIN would have quietly dropped any customer who's never ordered
anything, and that's exactly the customer management cares about seeing —
so this is a LEFT JOIN instead, keeping every customer row even when there's
no matching order.

```sql
SELECT c.customer_id, c.customer_name, o.order_id, o.order_date
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
ORDER BY c.customer_id;
```

![Q3 result](screenshots/q3_left_join.png)

16 rows — and right at the bottom, Fabrice Rugamba shows up with `NULL` for
order_id and order_date. That's the point of the LEFT JOIN: he's a
registered customer who's never actually bought anything.

---

### Q4 — Customers who spent above the average customer spend
*(CTE, then filtered against an aggregate in the outer query)*

First I need each customer's total spend (quantity × price, summed across
everything they've ever ordered), and *then* I need to compare each of
those totals against the average of all of them. That's a two-step
calculation, which is exactly what a CTE is for — compute `customer_totals`
once, then filter against `AVG(total_spent)` from that same CTE in the
outer query.

```sql
WITH customer_totals AS (
    SELECT c.customer_id, c.customer_name,
           SUM(oi.quantity * p.price) AS total_spent
    FROM customers c
    JOIN orders o       ON c.customer_id = o.customer_id
    JOIN order_items oi ON o.order_id = oi.order_id
    JOIN products p     ON oi.product_id = p.product_id
    GROUP BY c.customer_id, c.customer_name
)
SELECT customer_id, customer_name, total_spent
FROM customer_totals
WHERE total_spent > (SELECT AVG(total_spent) FROM customer_totals)
ORDER BY total_spent DESC;
```

![Q4 result](screenshots/q4_cte_above_average.png)

3 customers clear the average: Uwitonze Peace (116.70), Ntwari Kenny
(81.80), and David Habimana (64.10).

---

### Q5 — Rank customers by total spend, highest first
*(Window function: `RANK() OVER (ORDER BY ...)`)*

Same `customer_totals` CTE as Q4, but instead of filtering, I rank
everyone with `RANK()` so management can see the whole pecking order in one
shot, not just who's above average.

```sql
WITH customer_totals AS (
    SELECT c.customer_id, c.customer_name,
           SUM(oi.quantity * p.price) AS total_spent
    FROM customers c
    JOIN orders o       ON c.customer_id = o.customer_id
    JOIN order_items oi ON o.order_id = oi.order_id
    JOIN products p     ON oi.product_id = p.product_id
    GROUP BY c.customer_id, c.customer_name
)
SELECT customer_id, customer_name, total_spent,
       RANK() OVER (ORDER BY total_spent DESC) AS spend_rank
FROM customer_totals
ORDER BY spend_rank;
```

![Q5 result](screenshots/q5_rank.png)

Uwitonze Peace is #1, then Ntwari Kenny, David Habimana, Amini Niyonzima,
and Esther Ingabire last at 6.00 — she's only placed one order.

---

### Q6 — Number each customer's orders in the order they were placed
*(Window function: `ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date)`)*

This gives each customer their own 1, 2, 3... sequence of orders, reset for
every customer — that's what `PARTITION BY customer_id` does, as opposed to
one running count across the whole table.

```sql
SELECT customer_id, order_id, order_date,
       ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) AS order_sequence
FROM orders
ORDER BY customer_id, order_sequence;
```

![Q6 result](screenshots/q6_row_number.png)

Uwitonze Peace's 5 orders are numbered 1 through 5 in date order; every
other customer starts back at 1 for their own first order.

---

### Q7 — A running total of revenue over time
*(Window function: `SUM() OVER (ORDER BY order_date)`, i.e. a cumulative sum)*

Each order first has to be turned into a single revenue number
(`quantity × price` summed within that order — done in a CTE so a
multi-item order isn't double counted), and then `SUM() OVER (ORDER BY
order_date, order_id)` adds each order's revenue onto everything before it.

```sql
WITH order_revenue AS (
    SELECT o.order_id, o.order_date,
           SUM(oi.quantity * p.price) AS order_total
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    JOIN products p     ON oi.product_id = p.product_id
    GROUP BY o.order_id, o.order_date
)
SELECT order_id, order_date, order_total,
       SUM(order_total) OVER (ORDER BY order_date, order_id) AS running_revenue
FROM order_revenue
ORDER BY order_date, order_id;
```

![Q7 result](screenshots/q7_running_total.png)

Revenue climbs from 31.60 on the first order to 306.90 by the last one —
steadily, with no single order causing a huge jump.

---

### Q8 — Days between each customer's current and previous order
*(Window function: `LAG() OVER (PARTITION BY customer_id ORDER BY order_date)`)*

`LAG()` looks back one row within each customer's own order history and
pulls the previous order's date onto the current row, so `DATEDIFF()` can
turn that into a day count. Customers' *first* orders get filtered out with
`WHERE previous_order_date IS NOT NULL`, since there's nothing to compare a
first order to.

```sql
WITH customer_orders AS (
    SELECT customer_id, order_id, order_date,
           LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) AS previous_order_date
    FROM orders
)
SELECT customer_id, order_id, order_date, previous_order_date,
       DATEDIFF(order_date, previous_order_date) AS days_since_previous_order
FROM customer_orders
WHERE previous_order_date IS NOT NULL
ORDER BY customer_id, order_date;
```

![Q8 result](screenshots/q8_lag_days_between_orders.png)

10 rows — every customer with more than one order. Gaps range from 15 days
(Uwitonze Peace's first repeat purchase) up to 48 days (Amini Niyonzima
between her 1st and 2nd order). Esther Ingabire and Fabrice Rugamba don't
appear here at all — Esther has only 1 order, and Fabrice has none.

---

## What the results actually tell management

- **Uwitonze Peace is the standout customer** — top spender by a clear
  margin (116.70) and the most frequent buyer (5 orders), coming back
  roughly every 3–4 weeks. Alongside Ntwari Kenny and David Habimana, these
  three sit above the average customer spend (Q4/Q5) and are the obvious
  candidates for a loyalty programme or early access to promotions.
- **There's a customer who's never bought anything.** Fabrice Rugamba is in
  the system but has zero orders (Q3). A plain INNER JOIN would have hidden
  this completely — it's exactly the kind of gap a LEFT JOIN is meant to
  surface. A simple "welcome" discount could be worth testing on him.
- **How often customers come back varies a lot.** Gaps range from 15 to 48
  days (Q8). Esther Ingabire has ordered only once and sits at the bottom
  of the spend ranking (Q5) — she's a good target for a re-order reminder,
  since right now there's no sign she's coming back.
- **Revenue is growing steadily rather than in spikes.** The running total
  (Q7) climbs from 31.60 to 306.90 with no single order causing a big jump
  — the growth looks like it's coming from a broadening, repeat customer
  base rather than one-off large purchases, which is a healthier pattern.

## Challenges I ran into

- **Getting the LEFT JOIN in Q3 to actually mean something.** My first 5
  customers all had orders, so the LEFT JOIN behaved exactly like an INNER
  JOIN and proved nothing. I added a 6th customer, Fabrice Rugamba, and
  deliberately gave him no rows in `orders`.
- **The running total in Q7 needed two steps, not one.** Summing directly
  over `order_items` would have cumulatively summed at the item level and
  double-counted orders with more than one item. Aggregating to one row per
  order in a CTE first, *then* running the cumulative `SUM() OVER()` on top
  of that, fixed it.
- **`LAG()` returns NULL for a customer's very first order**, since there's
  nothing before it to compare to. Without filtering those out, Q8 would
  show a meaningless blank gap for every customer's first purchase — the
  `WHERE previous_order_date IS NOT NULL` clause takes care of it.
- **MySQL doesn't let you subtract two DATEs and get a day count** the way
  Oracle/Postgres do — I had to swap in `DATEDIFF(date1, date2)` for Q8
  instead of `date1 - date2`.
