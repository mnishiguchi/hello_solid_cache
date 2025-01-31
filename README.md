# Rails 8.0 Solid Cache Demo

This Rails 8.0 Solid Cache Demo showcases how to implement efficient fragment caching using Solid Cache with a dedicated PostgreSQL cache database. By following the provided instructions, you can set up the application, enable caching, verify its functionality, and manage the cache effectively using Docker Compose.

## Getting Started

Clone the Repository:

```bash
git clone https://github.com/mnishiguchi/hello_solid_cache.git
cd hello_solid_cache
```

Build the Docker images and set up the databases:

```bash
docker compose build
docker compose run --rm web bin/rails db:prepare
docker compose up
```

- `web` service: Rails development server accessible at http://localhost:3000.
- `db` service: PostgreSQL 17, managing both the primary and cache databases.

Navigate to http://localhost:3000/ to view the demo page.

On the first load, the page renders an "expensive block" that pauses for 2 seconds, simulating a resource-intensive operation.

Rails development mode disables caching by default. To enable it, execute the following command:

```bash
docker compose run --rm web rails dev:cache
```

This command creates a `tmp/caching-dev.txt` file, activating `config.action_controller.perform_caching = true`.

_Run the same command again to disable caching._

Press F5 or click the refresh button in your browser to reload http://localhost:3000/.

On subsequent loads, the "expensive block" should load instantly without the 2-second delay, displaying the same timestamp:

```
Expensive block rendered at: 2025-01-31 12:34:56 UTC
We paused for 2 seconds before rendering this the first time.
```

The timestamp remains unchanged, confirming that the content was fetched from the cache.

:tada:

---

## Inspect Rails Logs

To verify cache operations, review the Rails logs:

```bash
docker compose logs -f web
```

Sample Log Output:

```plaintext
web-1  | Started GET "/" for 172.18.0.1 at 2025-01-31 11:10:34 +0000
web-1  | Cannot render console from 172.18.0.1! Allowed networks: 127.0.0.0/127.255.255.255, ::1
web-1  | Processing by DemosController#show as HTML
web-1  |   Rendering layout layouts/application.html.erb
web-1  |   Rendering demos/show.html.erb within layouts/application
web-1  |   SolidCache::Entry Load (38.9ms)  SELECT "solid_cache_entries"."key", "solid_cache_entries"."value" FROM "solid_cache_entries" WHERE "solid_cache_entries"."key_hash" IN (-3304030544697551614) /*action='show',application='HelloSolidCache',controller='demos'*/
web-1  |   ↳ app/views/demos/show.html.erb:18
web-1  | Read fragment views/demos/show:df571e9d06a444c650513b1dbb09311b/expensive_block (41.6ms)
web-1  |   SolidCache::Entry Upsert (3.5ms)  INSERT INTO "solid_cache_entries" ("key","value","key_hash","byte_size","created_at") VALUES ('\x646576656c6f706d656e743a76696577732f64656d6f732f73686f773a64663537316539643036613434346336353035313362316462623039333131622f657870656e736976655f626c6f636b', '\x001102000000000000f0bf0a0000000408492200063a06454620202020203c212d2d2053696d756c61746520616e20657870656e73697665206f7065726174696f6e202d2d3e0a202020203c64697620636c6173733d22702d342062672d626c75652d353020626f726465722d6c2d3420626f726465722d626c75652d34303020726f756e646564223e0a2020202020203c7020636c6173733d226d622d3120666f6e742d73656d69626f6c6420746578742d626c75652d373030223e0a2020202020202020457870656e7369766520626c6f636b2072656e64657265642061743a20323032352d30312d33312031313a31303a3336205554430a2020202020203c2f703e0a2020202020203c7020636c6173733d22746578742d736d20746578742d677261792d363030223e0a202020202020202057652070617573656420666f722032207365636f6e6473206265666f72652072656e646572696e672074686973207468652066697273742074696d652e0a2020202020203c2f703e0a202020203c2f6469763e0a', -3304030544697551614, 611, CURRENT_TIMESTAMP) ON CONFLICT ("key_hash") DO UPDATE SET "key"=excluded."key","value"=excluded."value","byte_size"=excluded."byte_size" RETURNING "id" /*action='show',application='HelloSolidCache',controller='demos'*/
web-1  |   ↳ app/views/demos/show.html.erb:18
web-1  | Write fragment views/demos/show:df571e9d06a444c650513b1dbb09311b/expensive_block (31.3ms)
web-1  |   Rendered demos/show.html.erb within layouts/application (Duration: 2085.5ms | GC: 0.0ms)
web-1  |   Rendered layout layouts/application.html.erb (Duration: 2097.9ms | GC: 0.0ms)
web-1  | Completed 200 OK in 2123ms (Views: 2025.3ms | ActiveRecord: 73.7ms (2 queries, 0 cached) | GC: 0.0ms)
```

### Cache Miss (Initial Load)

```plaintext
SolidCache::Entry Load (38.9ms)  SELECT ...
Read fragment views/demos/show:.../expensive_block (41.6ms)
SolidCache::Entry Upsert (3.5ms)  INSERT INTO ...
Write fragment views/demos/show:.../expensive_block (31.3ms)
```

The fragment was not found in the cache, so it was rendered and then stored in Solid Cache.

### Cache Hit (Subsequent Loads)

```plaintext
SolidCache::Entry Load (0.6ms)  SELECT ...
Read fragment views/demos/show:.../expensive_block (2.3ms)
```

The fragment was successfully retrieved from Solid Cache, resulting in a faster response without the 2-second delay.

---

## Inspect the Cache Database

To confirm that Solid Cache is storing cached fragments, inspect the `solid_cache_entries` table.

Access the Cache Database via psql:

```bash
docker compose exec db psql -U postgres -d hello_solid_cache_development_cache
```

List Tables:

```sql
\dt
```

Expected Output:

```
                  List of relations
  Schema |         Name          | Type  |  Owner
---------+-----------------------+-------+----------
 public  | ar_internal_metadata  | table | postgres
 public  | schema_migrations     | table | postgres
 public  | solid_cache_entries   | table | postgres
(3 rows)
```

View Cached Entries:

```sql
SELECT * FROM solid_cache_entries;
```

Sample Output:

```
 id | key        | value      | created_at           | key_hash          | byte_size
----+------------+------------+----------------------+-------------------+-----------
 1  | \x646576...| \x0011...  | 2025-01-31 12:34:56  | -3304030544697551614| 611
(1 row)
```

- Key: Hex-encoded cache key.
- Value: Binary data of the cached fragment.
- Created_at: Timestamp when the fragment was cached.
- Key_hash and Byte_size: Internal details used by Solid Cache for indexing and storage management.

Exit psql:

```sql
\q
```

## Clear the Cache

To test the caching behavior again, clear the cache using Rails Console:

```bash
docker compose exec web bin/rails console
```

```ruby
Rails.cache.clear
```
