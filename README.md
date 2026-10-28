# Emaan Electronics — Shop Management System

Django application for POS billing, inventory, purchases, customers, suppliers, expenses, profit and reports.
Default database is SQLite (local use); PostgreSQL is recommended for production.

## 1. Install (local)

```bash
cd emaan
python -m venv venv
source venv/bin/activate            # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env                # Windows: copy .env.example .env
```

Edit `.env` and set `DJANGO_SECRET_KEY` (generate one with the command shown inside the file).

## 2. Create the database

The repository ships without migration files, so generate them once, then apply them:

```bash
python manage.py makemigrations accounts shop
python manage.py migrate
```

Keep the generated `accounts/migrations/` and `shop/migrations/` folders; they are part of your project.

## 3. Create the first administrator

```bash
python manage.py createsuperuser
```

Superusers are always given the Administrator role. Add cashier and staff accounts later from the Users page
(new accounts default to Cashier, the least-privileged role).

## 4. Run

```bash
python manage.py runserver
```

Open http://127.0.0.1:8000 and log in. Use the language switch for Urdu/English.

## 5. Demo data (testing only)

```bash
python manage.py seed_demo      # realistic sample records, flagged as demo
python manage.py clear_demo     # deletes demo records and only demo records
```

You can also remove demo data from Settings > Remove demo data. Real records are never flagged as demo.
Remove the demo data before you start entering real business records.

## 6. Stock integrity check

```bash
python manage.py check_stock
```

Verifies every product's quantity equals the sum of its stock movements (and serial counts).

## 7. Backup and restore

```bash
python manage.py backup_db            # writes to BACKUP_DIR (default: backups/), keeps newest 30
python manage.py backup_db --keep 60
```

Each run saves the database (SQLite file, or `pg_dump` custom format for PostgreSQL) plus an archive of uploaded images.
Schedule it daily (cron on Linux, Task Scheduler on Windows) and copy the backups off the machine.

**Restore SQLite:** stop the app, copy the chosen `backups/emaan-<stamp>.sqlite3` over `db.sqlite3`, start the app.
**Restore PostgreSQL:**

```bash
dropdb emaan_electronics && createdb emaan_electronics
pg_restore -h 127.0.0.1 -U emaan -d emaan_electronics backups/emaan-<stamp>.dump
```

**Restore images:** `tar -xzf backups/media-<stamp>.tar.gz -C media/`

## 8. PostgreSQL

Create a database and user, then in `.env` set `DB_ENGINE=postgres` and `DB_NAME`, `DB_USER`, `DB_PASSWORD`,
`DB_HOST`, `DB_PORT`. Run `migrate` again.

## 9. Deployment (secure online hosting)

1. Use a Linux server with Python 3.11+, PostgreSQL and nginx (or your host's equivalent).
2. In `.env` set: `DJANGO_DEBUG=False`, a long random `DJANGO_SECRET_KEY`, `DJANGO_ALLOWED_HOSTS=yourdomain.com`,
   `DJANGO_CSRF_TRUSTED_ORIGINS=https://yourdomain.com`, `DB_ENGINE=postgres`, and real SMTP settings for password reset.
3. `python manage.py migrate` and `python manage.py collectstatic --noinput`.
4. Run with gunicorn: `gunicorn config.wsgi:application --bind 127.0.0.1:8000 --workers 3` (use systemd to keep it running).
5. Put nginx in front with a TLS certificate (e.g. Let's Encrypt/certbot), forwarding `X-Forwarded-Proto`.
   Serve `/media/` from nginx. Static files are served by WhiteNoise.
6. With DEBUG off the app enables HTTPS redirect, secure cookies and HSTS automatically.
7. Never commit `.env`. Restrict database access to the server, schedule `backup_db`, and keep the OS updated.

## 10. Testing checklist (manual)

Run this on demo data first.

**Sales and stock**
- Note a product's quantity. Sell 2 in POS with a 10% discount, cash, full payment. Quantity drops by 2; invoice shows correct totals.
- Sell with partial payment to a customer: the remaining balance appears on the invoice and in the customer's balance.
- Try selling more than the available stock: it must be refused.
- Return one item from that sale: stock rises by 1, refund recorded, profit reduced accordingly.

**Purchases and suppliers**
- Create a purchase for 10 units and confirm it: stock rises by 10. Record a part payment: supplier due decreases.
- Return some units to the supplier: stock falls, supplier balance adjusts.

**Customers**
- Record a repayment; the balance drops; open and print the statement.

**Expenses and profit**
- Add an expense. Net profit = gross profit - expenses, where gross profit uses purchase cost of items sold (not total sales).

**Reports**
- Open every report under Reports, change the date range, search, then print or export CSV.

**Permissions**
- Log in as a cashier: POS works; financial reports, settings, users and the audit log are blocked.

**Integrity**
- `python manage.py check_stock` reports no mismatches.
