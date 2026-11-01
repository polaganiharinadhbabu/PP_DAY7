# Day 7 — PostgreSQL Projects, MongoDB & Redis

**Name:** Polagani Harinadh Babu  
**Date:** 1 November 2026  
**Topic:** PostgreSQL, MongoDB, Redis, and Backend Integration

## Learning Objective

Design a practical Python backend using PostgreSQL for relational business data, MongoDB for document-oriented content, and Redis for caching, sessions, messaging, and coordination.

---

## Task A — E-commerce PostgreSQL API

### 1. Create Database Tables

```sql
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(120) NOT NULL UNIQUE
);

CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    category_id INTEGER NOT NULL
        REFERENCES categories(id) ON DELETE RESTRICT,
    sku VARCHAR(80) NOT NULL UNIQUE,
    name VARCHAR(150) NOT NULL,
    description TEXT,
    price NUMERIC(10, 2) NOT NULL CHECK (price >= 0),
    stock INTEGER NOT NULL CHECK (stock >= 0),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'pending',
    total_amount NUMERIC(10, 2) NOT NULL CHECK (total_amount >= 0),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER NOT NULL
        REFERENCES orders(id) ON DELETE CASCADE,
    product_id INTEGER NOT NULL
        REFERENCES products(id) ON DELETE RESTRICT,
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price NUMERIC(10, 2) NOT NULL CHECK (unit_price >= 0)
);
```

### 2. Product Filtering

```sql
SELECT id, name, price, stock
FROM products
WHERE is_active = TRUE
  AND (:category_id IS NULL OR category_id = :category_id)
  AND (:min_price IS NULL OR price >= :min_price)
  AND (:max_price IS NULL OR price <= :max_price)
  AND (:search IS NULL OR name ILIKE '%' || :search || '%')
ORDER BY created_at DESC
LIMIT :page_size
OFFSET (:page - 1) * :page_size;
```

### 3. Safe Stock Update

```sql
UPDATE products
SET stock = stock - :quantity
WHERE id = :product_id
  AND is_active = TRUE
  AND stock >= :quantity;
```

Check the affected-row count. If zero rows are updated, the requested stock is unavailable and checkout must not be confirmed.

### 4. Checkout Transaction

```sql
BEGIN;

-- Validate available stock.
-- Insert order and order items.
-- Decrement stock using a conditional UPDATE.
-- Check affected-row counts.

COMMIT;

-- If any step fails, use ROLLBACK.
```

The order, order items, and stock changes must succeed or fail as one transaction. Store the purchase-time unit price in each order item.

### 5. API Endpoints

- GET /products — List and filter products.
- GET /products/{id} — Retrieve a product.
- POST /products — Create a product.
- PUT /products/{id} — Update a product.
- POST /checkout — Place an order.

---

## Task B — MongoDB Blog API

### 1. Create a Post Document

```javascript
db.posts.insertOne({
    title: "Introduction to FastAPI",
    body: "Learning backend development",
    tags: ["python", "backend"],
    author: {
        id: 21,
        name: "Asha"
    },
    created_at: new Date()
});
```

### 2. MongoDB CRUD Operations

```javascript
// List posts
db.posts.find({});

// Find posts by tag
db.posts.find({ tags: "python" });

// Update a field
db.posts.updateOne(
    { title: "Introduction to FastAPI" },
    { $set: { published: true } }
);

// Remove an obsolete field
db.posts.updateOne(
    { title: "Introduction to FastAPI" },
    { $unset: { obsolete_field: "" } }
);

// Delete a post
db.posts.deleteOne({
    title: "Introduction to FastAPI"
});
```

### 3. API Endpoints

- POST /posts — Create a post.
- GET /posts — List posts with pagination and optional filters.
- GET /posts/{id} — Retrieve a post.
- POST /posts/{id}/comments — Add a comment.
- GET /tags/{tag}/posts — Retrieve posts by tag.

Validate incoming data and MongoDB ObjectIds before querying the database.

### 4. Embedding vs Referencing Comments

Embed a bounded set of comments when they are frequently read and updated with the post.

If comments grow without a practical limit, store them in a separate `comments` collection and reference the post using `post_id`. Add indexes on `post_id` and `created_at`.

### 5. Aggregation — Count Posts per Tag

```javascript
db.posts.aggregate([
    { $unwind: "$tags" },
    {
        $group: {
            _id: "$tags",
            post_count: { $sum: 1 }
        }
    },
    { $sort: { post_count: -1 } }
]);
```

### 6. Aggregation — Top 10 Products by Units Sold

```javascript
db.orders.aggregate([
    { $match: { status: "paid" } },
    { $unwind: "$items" },
    {
        $group: {
            _id: "$items.product_id",
            units: { $sum: "$items.quantity" }
        }
    },
    { $sort: { units: -1 } },
    { $limit: 10 }
]);
```

### 7. Compound Index

```javascript
db.posts.createIndex({
    tags: 1,
    created_at: -1
});
```

MongoDB Compass is used to explore documents, build aggregation pipelines, and inspect indexes.

Motor is identified in the notes as an asynchronous MongoDB driver. Beanie is identified as an asynchronous ODM for Python document models.

---

## Task C — Redis Integration

### 1. Product Cache with Five-Minute TTL

```python
async def get_product_list(redis, cache_key, load_from_postgres):
    cached_value = await redis.get(cache_key)

    if cached_value is not None:
        return cached_value

    value = await load_from_postgres()

    # serialize() represents application-specific serialization.
    serialized_value = serialize(value)

    await redis.set(cache_key, serialized_value, ex=300)

    return value
```

Example cache key:

```text
products:category=audio:min=500:max=5000:page=1:page_size=20
```

Include all parameters that affect the response, such as filters, page, and page size.

### 2. Invalidate Cache After an Update

```python
async def update_product(redis, update_in_database, cache_keys):
    await update_in_database()

    if cache_keys:
        await redis.delete(*cache_keys)
```

Invalidate or refresh all relevant product-list and product-detail cache keys after the database update. The application must define how it handles cache races and stale data.

### 3. Store a Session Hash with Expiry

```redis
HSET session:abc user_id 42 role customer
EXPIRE session:abc 1800
HGETALL session:abc
TTL session:abc
```

Use a random session identifier and an expiry. Never store raw passwords in the session.

### 4. Safe Redis Distributed Lock

```redis
SET lock:invoice:123 <unique-token> NX EX 30
```

- NX sets the key only if it does not exist.
- EX 30 sets a 30-second expiry.
- Use a unique token to identify the lock owner.
- Verify the token before releasing the lock.
- If a worker crashes, the lock expires automatically.

Never blindly delete a lock because another worker may have acquired it after the original lock expired. Use an atomic token-checked release operation.

### 5. Redis Data Structures

- Strings — Counters, cached JSON, and OTPs.
- Hashes — Session fields.
- Lists — Ordered sequences and queues.
- Sets — Unique users who liked a post.
- Sorted sets — Leaderboards ordered by score.
- Pub/Sub — Transient message broadcasting.
- Streams — Retained event logs with consumer groups and acknowledgement.

---

## Task D — Debugging and Design Discussion

### 1. Stock Race Condition

**Problem:** Two customers read the same stock value before either decrements it. Both may attempt to buy the final unit.

**Solution:** Use a transaction and a conditional stock update:

```sql
UPDATE products
SET stock = stock - :quantity
WHERE id = :product_id
  AND stock >= :quantity;
```

Check the affected-row count. If no row is updated, reject checkout. Do not rely only on a previous SELECT.

### 2. Unlimited MongoDB Comments Array

**Problem:** The document grows continuously as comments are added.

**Solution:** Store comments separately in a `comments` collection and reference the post through `post_id`. Index `post_id` and `created_at`.

### 3. Cache Key Omits Page Size

**Problem:** Different pagination requests may return the same cached response and show the wrong number of products.

**Solution:** Include page, page size, filters, and other result-changing parameters in the cache key. Invalidate or refresh the relevant cache keys after updates.

---

## Integrated Backend Mini Project

### Architecture

- PostgreSQL — Products, categories, customers, orders, order items, and inventory.
- MongoDB — Blog posts, comments, tags, and flexible content documents.
- Redis — Product cache, sessions, rate-limit counters, temporary locks, and transient messaging.

### Suggested FastAPI Folder Structure

```text
app/
├── main.py
├── api/
├── services/
├── repositories/
├── models/
└── core/
```

- `api/` — Routers and request/response schemas.
- `services/` — Business logic and workflows.
- `repositories/` — Database queries and data access.
- `models/` — SQLAlchemy and document models.
- `core/` — Settings, logging, and security.

### Reliability and Security

1. Reuse database clients and use connection pooling.
2. Set connection and operation timeouts.
3. Store credentials in environment-based secrets.
4. Use parameterised SQL or ORM expressions.
5. Validate MongoDB ObjectIds and query filters.
6. Allowlist sortable and filterable fields.
7. Apply authentication, authorisation, and rate limiting.
8. Never log passwords or tokens.
9. Handle Redis outages deliberately; safe product reads may fall back to PostgreSQL with timeouts and monitoring.
10. Use idempotency for retryable checkout or payment operations.

### Suggested Tests

1. Reject a product with a negative price.
2. Reject duplicate product SKUs or slugs.
3. Verify that checkout creates an order and reduces stock.
4. Verify that checkout rolls back when a required step fails.
5. Reject checkout when stock is insufficient.
6. Verify MongoDB tag filtering.
7. Verify Redis cache expiry and invalidation.

