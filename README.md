# YugabyteDB Single-Node PoC for Beginners (WSL + Podman)

A hands-on, copy-paste lab to learn **YugabyteDB** (a distributed, PostgreSQL-compatible database) on a single container.
You will install it, load sample data, run **CRUD**, explore the **CLI and web UI**, and then clean everything up.

> **Level:** Beginner / fresher  |  **Time:** about 30-45 minutes  |  **Cost:** free, runs on your laptop

---

## Table of Contents

1. [What you will learn](#1-what-you-will-learn)
2. [Architecture](#2-architecture)
3. [Simple analogy](#3-simple-analogy)
4. [Prerequisites](#4-prerequisites)
5. [Folder layout](#5-folder-layout)
6. [Step 1 - Create the helper scripts](#6-step-1---create-the-helper-scripts)
7. [Step 2 - Install and start YugabyteDB](#7-step-2---install-and-start-yugabytedb)
8. [Step 3 - Create the schema](#8-step-3---create-the-schema)
9. [Step 4 - Load sample data](#9-step-4---load-sample-data)
10. [Step 5 - CRUD operations](#10-step-5---crud-operations)
11. [Step 6 - Explore the CLI and UI](#11-step-6---explore-the-cli-and-ui)
12. [Useful day-to-day commands](#12-useful-day-to-day-commands)
13. [Cleanup](#13-cleanup)
14. [Troubleshooting](#14-troubleshooting)
15. [Glossary](#15-glossary)
16. [What next](#16-what-next)
17. [Author and license](#17-author-and-license)

---

## 1. What you will learn

- Run YugabyteDB in a **Podman** container on Windows (WSL2)
- Create tables with **primary keys, foreign keys, CHECK and UNIQUE constraints**
- Load sample data with `generate_series`
- Perform **Create, Read, Update, Delete, Upsert and Transactions**
- Use the `ysqlsh` command line and the **web UIs**
- Peek at how data is **sharded into tablets**
- Remove every object, script and folder when you are done

## 2. Architecture

```
  Windows 10/11
  └── WSL2 (Ubuntu)
      └── Podman
          └── container: yb-node1   (single node, replication factor = 1)
              ├── YB-Master    metadata: tables, tablet map            UI  :7000
              ├── YB-TServer   stores and serves data (tablets)        UI  :9000
              ├── YSQL         PostgreSQL-compatible API                    :5433
              ├── YCQL         Cassandra-compatible API                     :9042
              └── yugabyted    friendly dashboard                       UI  :15433
```

- A table is split into **tablets** (shards). The primary key decides which tablet a row lands in.
- With one node there is **one copy** of the data. This is fine for learning, but it is **not highly available**.
  A 3-node setup (where each tablet is copied three times) is the natural next step.

## 3. Simple analogy

Think of a **library with one branch**.

| Library | YugabyteDB |
|---|---|
| Book collection | Table |
| Shelf | Tablet (shard) |
| Sorting rule that picks the shelf | Primary key (hash) |
| The librarian at the desk | YSQL query layer |
| Catalog office | YB-Master |
| Only one copy of every book | Replication factor 1 (single node) |

When you later add two more branches that each hold a copy of every shelf, you get high availability.

## 4. Prerequisites

| Requirement | Notes |
|---|---|
| Windows 10/11 with **WSL2** and Ubuntu | Linux or macOS also works. Skip the WSL parts. |
| **Podman** | Installed below |
| **4 GB+ RAM** available to WSL | YugabyteDB is memory hungry |
| Free ports | 5433, 9042, 7000, 9000, 15433 |
| Internet | To pull the container image (a few GB) |

Optional but recommended: give WSL more memory. Create `%UserProfile%\.wslconfig` in Windows:

```ini
[wsl2]
memory=6GB
processors=4
```

Then run `wsl --shutdown` in PowerShell and reopen Ubuntu.

Install Podman and check it:

```bash
sudo apt update
sudo apt install -y podman
podman --version
```

Create the project folder:

```bash
mkdir -p ~/yugabyte-poc-single/{scripts,sql}
cd ~/yugabyte-poc-single
```

Tested environment (fill in your own versions):

| Item | Version |
|---|---|
| OS | Ubuntu on WSL2 |
| Podman | `podman --version` |
| YugabyteDB image | `podman images` |

## 5. Folder layout

```
yugabyte-poc-single/
├── README.md
├── scripts/
│   ├── 01_install.sh      start the container and create the poc database
│   ├── run_sql.sh         run a .sql file against the poc database
│   ├── ysql.sh            open an interactive SQL shell
│   └── 99_cleanup.sh      remove everything
└── sql/
    ├── 01_schema.sql      tables and constraints
    ├── 02_sample_data.sql 100 customers, 20 products, 300 orders
    └── 03_crud.sql        create / read / update / upsert / delete / transactions
```

## 6. Step 1 - Create the helper scripts

Copy and paste this whole block into your terminal once.

```bash
cd ~/yugabyte-poc-single

cat > scripts/01_install.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
IMAGE="${YB_IMAGE:-docker.io/yugabytedb/yugabyte:latest}"

podman run -d --name yb-node1 --hostname yb-node1 \
  -p 5433:5433 -p 9042:9042 -p 7000:7000 -p 9000:9000 -p 15433:15433 \
  "$IMAGE" bin/yugabyted start --background=false

echo "Waiting for YSQL to be ready ..."
for i in $(seq 1 60); do
  if podman exec yb-node1 bin/ysqlsh -h yb-node1 -c "select 1" >/dev/null 2>&1; then
    echo "YugabyteDB is ready."
    podman exec yb-node1 bin/ysqlsh -h yb-node1 -c "CREATE DATABASE poc;" || true
    echo "UI: http://localhost:15433"
    exit 0
  fi
  sleep 5
done
echo "Timed out. Check logs: podman logs yb-node1"
exit 1
EOF

cat > scripts/run_sql.sh <<'EOF'
#!/usr/bin/env bash
# usage: ./scripts/run_sql.sh sql/file.sql [database]
podman exec -i yb-node1 bin/ysqlsh -h yb-node1 -d "${2:-poc}" -a -f - < "$1"
EOF

cat > scripts/ysql.sh <<'EOF'
#!/usr/bin/env bash
podman exec -it yb-node1 bin/ysqlsh -h yb-node1 -d "${1:-poc}"
EOF

cat > scripts/99_cleanup.sh <<'EOF'
#!/usr/bin/env bash
read -r -p "Remove the container, image and ~/yugabyte-poc-single ? [y/N] " a
[[ "$a" == "y" ]] || { echo "Aborted"; exit 0; }
podman rm -f yb-node1 2>/dev/null
podman images --format '{{.ID}}' --filter reference='*yugabytedb/yugabyte*' | sort -u | xargs -r podman rmi -f
# Optional: uninstall podman itself
# sudo apt remove --purge -y podman && sudo apt autoremove -y
cd ~ && rm -rf ~/yugabyte-poc-single
echo "Cleanup complete."
EOF

chmod +x scripts/*.sh
```

Check:

```bash
ls -l scripts
```

All four files should be listed with `x` (executable) permission.

## 7. Step 2 - Install and start YugabyteDB

```bash
./scripts/01_install.sh
```

What happens:
1. Podman pulls the image (several minutes the first time).
2. The container `yb-node1` starts.
3. The script waits until YSQL answers, usually 1-3 minutes, then creates a database called `poc`.

Tip: to keep your lab reproducible, pin a version instead of `latest`:

```bash
YB_IMAGE=docker.io/yugabytedb/yugabyte:<tag> ./scripts/01_install.sh
```

Verify:

```bash
podman ps
podman exec yb-node1 bin/yugabyted status
```

Open these in your Windows browser:

| UI | URL | What it shows |
|---|---|---|
| yugabyted dashboard | http://localhost:15433 | Overview, databases, performance, queries |
| Master UI | http://localhost:7000 | Tables and tablets, cluster metadata |
| TServer UI | http://localhost:9000 | Tablets hosted on this server, RPCs |

## 8. Step 3 - Create the schema

```bash
cat > sql/01_schema.sql <<'EOF'
\timing on
DROP TABLE IF EXISTS orders, products, customers CASCADE;

CREATE TABLE customers (
  customer_id INT PRIMARY KEY,
  full_name   TEXT NOT NULL,
  email       TEXT NOT NULL UNIQUE,
  city        TEXT,
  created_at  TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE products (
  product_id INT PRIMARY KEY,
  name       TEXT NOT NULL,
  category   TEXT,
  price      NUMERIC(10,2) NOT NULL CHECK (price > 0)
);

CREATE TABLE orders (
  order_id    INT PRIMARY KEY,
  customer_id INT NOT NULL REFERENCES customers(customer_id),
  product_id  INT NOT NULL REFERENCES products(product_id),
  qty         INT NOT NULL CHECK (qty > 0),
  status      TEXT NOT NULL DEFAULT 'NEW' CHECK (status IN ('NEW','PAID','SHIPPED','CANCELLED')),
  order_ts    TIMESTAMPTZ DEFAULT now()
);

\dt
EOF

./scripts/run_sql.sh sql/01_schema.sql
```

Expected: three `CREATE TABLE` lines, then a list with `customers`, `orders`, `products`.
A "table does not exist, skipping" notice on the first run is normal.

## 9. Step 4 - Load sample data

```bash
cat > sql/02_sample_data.sql <<'EOF'
\timing on
INSERT INTO customers (customer_id, full_name, email, city)
SELECT g, 'Customer '||g, 'cust'||g||'@example.com',
       (ARRAY['Chennai','Bengaluru','Mumbai','Delhi','Madurai'])[1 + g % 5]
FROM generate_series(1,100) g;

INSERT INTO products (product_id, name, category, price)
SELECT g, 'Product '||g,
       (ARRAY['Books','Electronics','Grocery','Toys'])[1 + g % 4],
       round((10 + (g*13 % 190))::numeric, 2)
FROM generate_series(1,20) g;

INSERT INTO orders (order_id, customer_id, product_id, qty, status)
SELECT g, 1 + (g % 100), 1 + (g % 20), 1 + (g % 5),
       (ARRAY['NEW','PAID','SHIPPED','CANCELLED'])[1 + g % 4]
FROM generate_series(1,300) g;

SELECT 'customers' AS tbl, count(*) FROM customers
UNION ALL SELECT 'products', count(*) FROM products
UNION ALL SELECT 'orders',   count(*) FROM orders;
EOF

./scripts/run_sql.sh sql/02_sample_data.sql
```

Expected counts: **customers 100, products 20, orders 300**.

## 10. Step 5 - CRUD operations

```bash
cat > sql/03_crud.sql <<'EOF'
\echo '=== CREATE ==='
INSERT INTO customers (customer_id, full_name, email, city)
VALUES (101, 'Clement Test', 'clement@example.com', 'Bengaluru') RETURNING *;

INSERT INTO orders (order_id, customer_id, product_id, qty)
VALUES (1001, 101, 5, 2) RETURNING *;

\echo '=== READ ==='
SELECT * FROM customers WHERE customer_id = 101;
SELECT city, count(*) AS customers FROM customers GROUP BY city ORDER BY customers DESC;
SELECT c.full_name, p.name AS product, o.qty, round(o.qty * p.price, 2) AS amount, o.status
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
JOIN products  p ON p.product_id  = o.product_id
WHERE o.customer_id = 101;

\echo '=== UPDATE ==='
UPDATE customers SET city = 'Chennai' WHERE customer_id = 101 RETURNING *;
UPDATE orders SET status = 'PAID' WHERE order_id = 1001 RETURNING *;

\echo '=== UPSERT (insert or update) ==='
INSERT INTO customers (customer_id, full_name, email, city)
VALUES (101, 'Clement Upserted', 'clement@example.com', 'Madurai')
ON CONFLICT (customer_id) DO UPDATE
  SET full_name = EXCLUDED.full_name, city = EXCLUDED.city
RETURNING *;

\echo '=== TRANSACTION (commit) ==='
BEGIN;
  INSERT INTO orders (order_id, customer_id, product_id, qty) VALUES (1002, 101, 7, 1);
  UPDATE orders SET status = 'SHIPPED' WHERE order_id = 1001;
COMMIT;
SELECT order_id, status FROM orders WHERE customer_id = 101 ORDER BY order_id;

\echo '=== TRANSACTION (rollback) ==='
BEGIN;
  DELETE FROM orders WHERE customer_id = 101;
ROLLBACK;
SELECT count(*) AS orders_still_there FROM orders WHERE customer_id = 101;

\echo '=== CONSTRAINT CHECKS (these three SHOULD fail) ==='
INSERT INTO products VALUES (999, 'Bad', 'Toys', -5);
INSERT INTO customers (customer_id, full_name, email) VALUES (102, 'Dup', 'cust1@example.com');
INSERT INTO orders (order_id, customer_id, product_id, qty) VALUES (1003, 99999, 1, 1);

\echo '=== DELETE (child rows first because of the foreign key) ==='
DELETE FROM orders    WHERE customer_id = 101;
DELETE FROM customers WHERE customer_id = 101;
SELECT count(*) AS remaining FROM customers WHERE customer_id = 101;
EOF

./scripts/run_sql.sh sql/03_crud.sql
```

What to look for:

| Section | Expected result |
|---|---|
| CREATE / UPDATE / UPSERT | The row is echoed back by `RETURNING *` |
| Transaction (commit) | Orders 1001 and 1002 exist, 1001 is `SHIPPED` |
| Transaction (rollback) | `orders_still_there` is greater than 0, because the delete was undone |
| Constraint checks | Three errors: check constraint, unique constraint, foreign key constraint |
| DELETE | `remaining = 0` |

## 11. Step 6 - Explore the CLI and UI

Open an interactive SQL shell:

```bash
./scripts/ysql.sh
```

Try these inside `ysqlsh` (type `\q` to quit):

```sql
\dt                                  -- list tables
\d orders                            -- describe a table
\timing on                           -- show query time
SELECT * FROM customers LIMIT 5;
SELECT * FROM yb_servers();          -- nodes in the cluster
SELECT * FROM yb_table_properties('customers'::regclass);   -- tablets and sharding info
EXPLAIN (ANALYZE, DIST, COSTS OFF) SELECT * FROM customers WHERE customer_id = 42;
```

In the `EXPLAIN` output, look at **Storage Table Read Requests**. A lookup by primary key needs very few reads, while a full scan reads more.

UI checklist:
1. http://localhost:15433 -> **Databases** -> `poc`
2. http://localhost:7000 -> **Tables** -> click `customers` to see its tablet(s)
3. http://localhost:9000 -> **Tablets**

Optional: the Cassandra-compatible API (YCQL).

```bash
podman exec -it yb-node1 bin/ycqlsh yb-node1
```

```sql
CREATE KEYSPACE demo;
USE demo;
CREATE TABLE t (id int PRIMARY KEY, name text);
INSERT INTO t (id, name) VALUES (1, 'hello');
SELECT * FROM t;
```

## 12. Useful day-to-day commands

```bash
podman ps                     # is it running?
podman stop yb-node1          # stop (data is kept inside the container)
podman start yb-node1         # start again (wait ~1 minute)
podman logs -f yb-node1       # follow logs
podman exec -it yb-node1 bash # shell inside the container
```

Data lives **inside the container**. Stopping and starting keeps it. Removing the container (or running the cleanup script) deletes it.

## 13. Cleanup

When you have finished, remove the container, the image and this whole folder:

```bash
./scripts/99_cleanup.sh       # answer y
```

Verify:

```bash
podman ps -a
podman images
ls ~/yugabyte-poc-single      # should say: No such file or directory
```

## 14. Troubleshooting

| Symptom | Fix |
|---|---|
| `Permission denied` running a script | `chmod +x scripts/*.sh` |
| Install script times out | Run `podman logs yb-node1`. Low memory is the usual cause, so give WSL at least 4 GB (see Prerequisites). |
| `container name "yb-node1" is already in use` | `podman rm -f yb-node1`, then run the install again |
| `port is already allocated` | Another program uses one of the ports. Find it with `ss -ltn`. |
| `ysqlsh` cannot connect | Use `-h yb-node1` (the scripts already do). `localhost` often fails inside the container. |
| SQL file errors about a missing table | Run the steps in order: schema, sample data, CRUD |
| Browser cannot open `localhost:15433` | Check `podman ps` shows the port mappings. Try `wsl --shutdown` from PowerShell and restart. |

Some SQL syntax and `yb_*` helper functions can differ slightly between YugabyteDB releases. If something fails, check your version with `SELECT version();` and consult the official docs.

## 15. Glossary

| Term | Meaning |
|---|---|
| **YSQL** | YugabyteDB's PostgreSQL-compatible SQL API |
| **YCQL** | YugabyteDB's Cassandra-compatible API |
| **Tablet** | A shard: a slice of a table stored and replicated as a unit |
| **YB-Master** | Service that keeps cluster metadata |
| **YB-TServer** | Service that stores tablets and serves reads and writes |
| **Raft** | Consensus protocol used to keep tablet replicas in sync (important in multi-node setups) |
| **RF (replication factor)** | Number of copies of each tablet. This lab uses 1. |
| **Podman** | Daemonless container engine, a drop-in alternative to Docker |

## 16. What next

Follow-up labs (multi-node):
1. 3-node cluster, with replication, tablet leaders and node hops
2. High availability and zero data loss (stop a node, keep working)
3. Linear write scaling (add nodes)
4. Real-time sharding and data distribution visuals
5. Drop-in PostgreSQL application compatibility

## 17. Author and license

- GitHub: [softclement](https://github.com/softclement)
- Contributions and issues are welcome.
- License: MIT. Add a `LICENSE` file to your repository.

> This is a learning lab, not a production setup. A single node has no redundancy.
