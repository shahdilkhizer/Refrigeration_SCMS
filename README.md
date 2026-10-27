# Refrigeration Gas Cylinder Supply Chain Management System

A web application that tracks refrigerant gas cylinders from the imported container to the customer's invoice.

**Contents**

1. [What this project is](#1-what-this-project-is)
2. [The business, step by step](#2-the-business-step-by-step)
3. [Login and who can do what](#3-login-and-who-can-do-what)
4. [How the system is built](#4-how-the-system-is-built)
5. [The database and the ERDs](#5-the-database-and-the-erds)
6. [How to run it](#6-how-to-run-it)
7. [Tests](#7-tests)
8. [Backup and moving to another computer](#8-backup-and-moving-to-another-computer)
9. [Project folders](#9-project-folders)

---

## 1. What this project is

A company in Pakistan buys refrigerant gas from factories abroad, mostly China. The gas is the
one inside fridges and air conditioners (R-134a, R-410a, R-22, R-32, R-404a) and it comes in
steel cylinders packed in shipping containers. The containers arrive at Karachi port. From
there the cylinders are sold to dealers or sent by truck to the company's depots in Lahore,
Islamabad and Multan.

This system keeps track of every cylinder from the day the container is booked until the day
the cylinder is sold. For any invoice it can tell you which batch the cylinders came from, which
container carried them and which foreign supplier made them. It also keeps the supplier's invoice
and the payment the company made to the supplier, in US dollars.

It is built so that three things are always true:

* **Oldest stock leaves Karachi first** (FIFO, explained in section 2.4), whether it is sold or sent to a depot. Nobody can pick a batch by hand.
* **Stock never goes below zero**, and a half finished order never leaves a mess behind.
* **Everything can be traced** from an invoice back to the supplier, and every important action is written in an audit log.

---

## 2. The business, step by step

This section is written for someone who has never seen the software. You do not need to know anything about computers to follow it.

### 2.1 The company in one paragraph

The company imports cylinders and keeps them in a big warehouse in Karachi, the **Central Hub**. Three smaller warehouses, the **regional depots**, are in Lahore, Islamabad and Multan. Big dealers buy straight from Karachi. Smaller shops in the other cities buy from the depot nearest to them. Stock reaches a depot by truck from Karachi.

### 2.2 The people who use the system

| Person | What they do in real life | What they do in the system |
| :-- | :-- | :-- |
| **System Admin** | Looks after the company records, the staff and the payments | Keeps the lists of vendors, customers, products and warehouses. Records the payment to a vendor. Creates users and decides what each user may see. Reads the audit log. |
| **Central Logistics Manager** | Handles imports and the Karachi warehouse | Books shipments, marks them cleared by customs, receives them at the dock, sends stock to depots. Can see what is billed and paid for each shipment. |
| **Regional Warehouse Supervisor** | Runs one depot (for example Lahore) | Counts the truck when it arrives, confirms it, and sells from the depot stock. Sees only their own depot. |
| **Sales Officer** | Takes orders from customers | Creates sales orders and prints the invoice and the delivery challan. |

### 2.3 The journey of a cylinder

![The business process](docs/process_flow.svg)

**Step 1. Buying (Book the import).**
The logistics manager records the shipment: the supplier, the **Bill of Lading** (the shipping company's number for the cargo), the container number, the port arrival date, how many cylinders of each gas were ordered, and the **supplier's invoice** (its number and the amount in US dollars).

**Step 2. Customs.**
When the container reaches the port it must pass customs. Once the papers are fine, the logistics manager marks the shipment **Cleared**. Until then the system refuses to receive any stock from it. This follows the rule that stock may only be added from a cleared shipment.

**Step 3. Unloading and checking (Receive at the dock).**
Staff count the cylinders and inspect them. For every line they enter the counted quantity and the **bay** (the shelf position in the warehouse).

* If the cylinders pass, a **batch** is created in the Karachi stock. The system writes down the exact arrival time of the batch.
* If they fail the inspection they are put in **Quarantine**. They stay in the warehouse but are never sold or sent anywhere.
* The system refuses a count larger than what was ordered, and refuses stock that would not fit in the warehouse (every warehouse has a capacity).

**Step 4. Paying the vendor.**
The company pays the foreign supplier itself, in US dollars, by telegraphic transfer (T/T), letter of credit (L/C) or bank draft, as agreed in the vendor's **billing terms** (for example "T/T within 30 days of the B/L date"). The System Admin records the payment on the shipment: the date, how it was paid and the bank's reference number. The payment is always the **whole invoice** in one go, and a shipment can be paid only once. Until then the shipment shows **Unpaid**, and the Dashboard and the Shipments report show how much is still to be paid to vendors.

**Step 5. Selling from Karachi.**
A customer asks for cylinders. The sales officer chooses the customer, the gases, the quantities and the delivery date. The system then checks three things: there is enough stock, the order is not above the customer's **credit limit**, and the delivery date is not in the past. If all is well, the system takes the cylinders from the oldest batches first, fixes the price at that moment, and makes an **invoice**: the value of the cylinders, plus **18% GST**, gives the net payable. A printable invoice and a delivery challan are available at once.

**Step 6. Sending stock to a depot (Transfer).**
The logistics manager chooses the depot and enters the truck's number plate. The system picks the oldest cylinders by itself. The stock is now **In-Transit**: it has left Karachi but it is not in the depot yet, so nobody can sell it.

**Step 7. Receiving at the depot.**
When the truck arrives, the depot supervisor counts the cylinders and types in the number that really arrived. Only then does the depot stock go up, and it goes up by the counted number. If 40 were sent and 35 arrived, the depot gets 35 and the system records that 5 were lost on the road.

**Step 8. Selling from a depot.**
The depot supervisor sells from the depot stock using the same order screen. They can only choose their own depot.

**Step 9. Looking back.**
* **Track Order** starts from an invoice and shows the batch, the arrival date, the Bill of Lading, the supplier's invoice and the foreign supplier of every cylinder sold (and whether that supplier invoice is paid, for users who may see payments).
* **Reports** show stock value, sales, shipments and transfers. Each user sees only the reports the admin has allowed.
* The **Audit Log** (admin only) lists who did what and when, including failed logins.

### 2.4 FIFO in plain words

FIFO means **First In, First Out**. It works like a shelf of milk in a shop: the packets that came first are at the front and are sold first, so nothing stays at the back for months.

An example. Karachi holds two batches of the same gas:

| Batch | Arrived | Cylinders |
| :-- | :-- | :-: |
| #1 | 12 September | 120 |
| #5 | 7 October | 600 |

A customer orders 130 cylinders. The system takes all 120 from batch #1 (now empty), and the other 10 from batch #5 (590 left). Both pieces are recorded, so later we know exactly which batch each cylinder came from. A person cannot change this choice, on purpose.

### 2.5 Rules the system enforces

* Stock can only be received from a shipment that customs has cleared.
* A count cannot be more than what was ordered. Nothing can be stored beyond the warehouse capacity.
* Failed cylinders go to Quarantine and are never sold.
* A shipment is booked with the supplier's invoice number and an invoice amount above 0, in US dollars.
* A vendor is paid the full invoice at once, in one payment, with a bank reference. A second payment for the same shipment is refused, and the payment date cannot be in the future.
* Only users with the "Record a payment" permission can pay a vendor, and only users with "View invoice amounts and payments" can see the amounts at all.
* An order needs enough stock, a delivery date that is not in the past, and a total (with tax) within the customer's credit limit.
* Every invoice carries 18% GST. The rate is saved on the invoice, so an old invoice never changes.
* One order has exactly one invoice.
* A transfer needs a truck number plate. Stock reaches the depot only after the supervisor confirms it, and only by the counted number.
* A customer, product, vendor or warehouse that is already used cannot be deleted.
* If any step of an order or a transfer fails, nothing at all is saved.

### 2.6 Words you will meet

| Word | Meaning |
| :-- | :-- |
| Vendor | The foreign company that supplies the cylinders |
| Bill of Lading (B/L) | The shipping company's document and number for a cargo |
| Supplier invoice | The bill the foreign vendor sends for a shipment, in US dollars |
| Billing terms | How and when the vendor wants to be paid, for example "T/T within 30 days of B/L date" |
| T/T, L/C | Telegraphic transfer (a bank transfer) and letter of credit (the bank promises to pay the vendor) |
| Container | The big steel box that carries the cylinders by sea |
| Customs clearance | Permission from customs to take the cargo out of the port |
| Batch | One lot of cylinders received together, with one arrival time |
| Bay | The shelf position of a batch inside the warehouse |
| Quarantine | A batch that failed inspection and must not be sold |
| Central Hub / Depot | The main warehouse in Karachi / a smaller warehouse in another city |
| Transfer | Moving stock from Karachi to a depot by truck |
| In-Transit | On the truck, not yet counted at the depot |
| Invoice | The bill for an order |
| Challan | The delivery note that goes with the cylinders |
| GST | The sales tax, 18% here |
| Credit limit | The largest single order a customer may place |
| Tare weight | The weight of the empty cylinder |

---

## 3. Login and who can do what

Everyone logs in with a user name and a password. After login the menu shows only the tabs that user is allowed to open. If someone types the address of a page they may not use, they get a "no access" page, and the server refuses the request as well.

**Demo users** (created by `seed_data.sql`):

| User name | Password | Role |
| :-- | :-- | :-- |
| admin | admin123 | System Admin |
| logistics | logistics123 | Central Logistics Manager |
| lahore | lahore123 | Regional Warehouse Supervisor (Lahore depot) |
| multan | multan123 | Regional Warehouse Supervisor (Multan depot) |
| sales | sales123 | Sales Officer |

Passwords are stored as PBKDF2 hashes, never as plain text.

### 3.1 Default access

When the database is created, every user gets the default access of their role. A tick means the user has it. The System Admin always has everything.

| Module | Permission | Admin | Logistics | Supervisor | Sales |
| :-- | :-- | :-: | :-: | :-: | :-: |
| Customers | view | yes | | yes | yes |
| | add, edit, delete | yes | | | |
| Vendors | view | yes | yes | | |
| | add, edit, delete | yes | | | |
| Products | view | yes | yes | yes | yes |
| | add, edit, delete | yes | | | |
| Warehouses | view | yes | yes | yes | |
| | add, edit, delete | yes | | | |
| Shipments | view | yes | yes | | |
| | book, mark customs cleared | yes | yes | | |
| | receive at dock | yes | yes | | |
| Vendor Payments | view invoice amounts and payments | yes | yes | | |
| | record a payment | yes | | | |
| FIFO Inventory | view | yes | yes | | yes |
| Sales Orders | view | yes | | yes | yes |
| | create order | yes | | yes | yes |
| Transfers | view | yes | yes | yes | |
| | send transfer | yes | yes | | |
| | confirm receipt | yes | | yes | |
| Track Order | view | yes | yes | yes | yes |
| Reports | stock valuation | yes | yes | | |
| | sales | yes | | | yes |
| | shipments | yes | yes | | |
| | transfers | yes | yes | | |
| Users & Access | everything | yes | | | |
| Audit Log | view | yes | | | |

Users & Access and Audit Log belong to the System Admin only. They cannot be given to anyone else.
The Dashboard is open to every logged in user. A depot supervisor sees only the figures of their own depot.

### 3.2 Changing access (admin only)

Open **Users & Access**. The list of users is on the left. Click a user to see their details and a grid of all permissions, grouped by module. Reports are listed one by one (stock, sales, shipments, transfers), so the admin decides which report each user can see.

* Tick or untick the permissions and press **Save access**.
* **Reset to role defaults** puts the user back to the table above.
* **Save details** changes the name, the role, the depot or switches the user off. **Change password** sets a new password.
* **New User** creates a login. A new user starts with the default access of the chosen role.

Every change is saved in the database (table `User_Access`) and is written to the audit log. The server reads the access from the database on every request, so a change works from the user's very next click, and it is still there after a restart.

The admin cannot switch off or demote their own account, and the last active admin cannot be removed.

**To see it work:** log in as `sales`, note the tabs, log in as `admin`, remove or add a permission for `sales` (for example give `customers.edit`, or take away `reports.sales`), then log in as `sales` again and the menu has changed.

---

## 4. How the system is built

![System architecture](docs/architecture.svg)

The system has three parts.

| Part | What it is | Folder |
| :-- | :-- | :-- |
| Web app | Next.js 14, React 18, TypeScript, Tailwind CSS. What the user sees in the browser. | `frontend/` |
| API server | Django 6 and Django REST Framework (Python). Checks who the user is, checks the rules, talks to the database. | `backend/` |
| Database | PostgreSQL 18. Holds all the data in 18 tables. | `backend/schema.sql` |

The Django models use `managed = False`. The SQL file, not Django, creates and owns the tables.

### 4.1 What happens at login

1. The user types a user name and a password on the login page.
2. The server finds the user in `App_Users` and compares the password with the stored hash.
3. If it matches, the server sends back a token (a signed text valid for 8 hours) together with the user's role and permissions. A wrong password or a switched-off account is refused, and the attempt goes to the audit log.
4. The browser keeps the token and sends it with every later request.

### 4.2 What happens on every request

1. The server reads the token and loads the user from the database.
2. It looks up the permission that this URL needs (for example `orders.create`) in `User_Access`.
3. No permission means HTTP 403 and nothing is done. Otherwise the view runs.

### 4.3 What happens when an order is saved

`POST /api/orders/` with the customer, the delivery date and the lines.

1. The access check above.
2. The server checks the customer, the products, the quantities and the delivery date.
3. Before saving anything it adds up the lines, works out the 18% tax and the net payable, and compares the net payable with the customer's credit limit. Over the limit means the order is refused.
4. Everything after this happens in one database transaction. The order and its lines are written with the price copied from the product.
5. For an order from Karachi, the server locks the open batches of that warehouse (so two people cannot take the same cylinders), reads them in order of `arrival_timestamp`, takes from the oldest first, and writes one `Dispatch_Allocations` row for every batch it used. For an order from a depot, the depot's number in `Regional_Inventory` is reduced.
6. The invoice is written with total, tax and net payable. If any step fails (for example not enough stock on the second line), the whole transaction is rolled back and nothing is saved.
7. The audit log entry is written and the new order is returned.

The same pattern is used for the dock receipt and for the transfers. The code is in `backend/supply_chain/services/fifo_engine.py`.

### 4.4 Where the data is kept

All data lives in the PostgreSQL database on the computer, not in the browser and not in files of the project. The PostgreSQL Windows service starts with Windows, so the data is there the next time the computer is on. See section 8 to copy the data to another computer.

---

## 5. The database and the ERDs

18 tables in 6 modules. The pictures are generated from `backend/schema.sql` by `docs/erd/build_erd.py`, so they always match the real tables. Open any `.svg` in a browser to see it bigger. `docs/erd/print.html` has all of them, one per landscape page, for printing.

**How to read the lines** (crow's foot notation): a single bar means "exactly one", a circle means "zero", a three-pronged foot means "many".
`PK` is a primary key, `FK` a foreign key, `UQ` a unique column.

### 5.1 The complete ERD (names only)

All 18 tables on one page. The colours are the modules below.

![Complete ERD](docs/erd/overview.svg)

### 5.2 Module 1: Master Data

The four registers that everything else points to. Customers, vendors, products and warehouses are kept by the admin.

![Master Data](docs/erd/module_1_master_data.svg)

### 5.3 Module 2: Inbound Shipments and Vendor Payments

A container is one row in `Import_Shipments`, together with the supplier's invoice number and amount (US dollars). Each product in it is a row in `Shipment_Items`. Customs status decides whether it can be received. When the company pays the vendor, one row is written in `Vendor_Payments`. The link is one to zero-or-one: a shipment that is not paid yet has no payment row, and it can never have two, because the whole invoice is paid at once.

![Inbound Shipments and Vendor Payments](docs/erd/module_2_inbound_shipments.svg)
### 5.4 Module 3: Inventory and FIFO

One row of `Inventory_Batches` for every lot received at the dock. `arrival_timestamp` is what FIFO sorts on. `Dispatch_Allocations` records which order line took how many cylinders from which batch, and this is what makes tracing possible.

![Inventory and FIFO](docs/erd/module_3_inventory_fifo.svg)

### 5.5 Module 4: Sales and Invoicing

An order has a header (`Sales_Orders`) and lines (`Order_Items`). `Invoices.order_id` is unique, so an order can have only one invoice. `net_payable = total_amount + tax_amount` is checked by the database.

![Sales and Invoicing](docs/erd/module_4_sales_invoicing.svg)

### 5.6 Module 5: Transfers and Depot Stock

A truck is one row in `Stock_Transfers`, and its cargo is `Transfer_Items` (one per batch used). `received_qty` is the number counted at the depot. `Regional_Inventory` is the running stock of each depot.

![Transfers and Depot Stock](docs/erd/module_5_transfers_depot_stock.svg)

### 5.7 Module 6: Security and Audit

Users, what each user may do, and the log of what happened.

![Security and Audit](docs/erd/module_6_security_audit.svg)

### 5.8 Table list

| Table | Module | What it holds |
| :-- | :-- | :-- |
| Vendors | Master Data | Foreign suppliers and their billing terms |
| Customers | Master Data | Buyers, their type and credit limit |
| Products | Master Data | Gas type, cylinder size, tare weight, price |
| Warehouses | Master Data | Karachi hub and the depots, with capacity |
| Import_Shipments | Inbound | One row per container: B/L, container number, customs status, supplier invoice number and amount (USD) |
| Shipment_Items | Inbound | Products in a container: ordered and received quantity |
| Vendor_Payments | Inbound | The payment of a supplier invoice: amount (USD), date, method, bank reference, who recorded it |
| Inventory_Batches | Inventory and FIFO | One row per received lot, with arrival time, bay and quantity left |
| Dispatch_Allocations | Inventory and FIFO | Which order line took how many cylinders from which batch |
| Sales_Orders | Sales | Order header: customer, warehouse, delivery date |
| Order_Items | Sales | Order lines with the price fixed at order time |
| Invoices | Sales | One invoice per order: total, tax rate, tax, net payable |
| Stock_Transfers | Transfers | One row per truck: from, to, truck plate, status |
| Transfer_Items | Transfers | Batches on the truck, sent and counted quantity |
| Regional_Inventory | Transfers | Current stock of each depot per product |
| App_Users | Security | Logins, roles, the depot of a supervisor |
| User_Access | Security | One row for each permission a user has |
| Audit_Logs | Security | Who did what and when |

The design follows the **header and lines** pattern three times (shipments, orders, transfers), so one container, one order or one truck can hold many products without repeating columns.

`docs/ERD_CHANGES.md` lists exactly what is new compared with the first 14 table ERD (including the vendor payment), and `docs/REQUIREMENTS_TRACE.md` links every SRS and SDS item to the table or screen that covers it.

---

## 6. How to run it

You need PostgreSQL 18, Python 3 and Node.js.

**First time on a new computer**

1. In pgAdmin create an empty database called `refrigeration_scms_db`. The user is `postgres` and the password is set in `backend/scms_backend/settings.py`.
2. Run `backend/schema.sql`, then `backend/seed_data.sql` on that database (pgAdmin Query Tool, or `psql`).
   Warning: `schema.sql` starts by dropping the tables, so it deletes the data of an existing database. Do not run it on a database you want to keep.
3. Backend packages:
   ```
   cd backend
   python -m venv venv
   venv\Scripts\pip install -r requirements.txt
   ```
4. Frontend packages:
   ```
   cd frontend
   npm install
   ```

If the data is coming from another computer, skip step 2 and use `restore_db.bat` (section 8).

**Every day**

Double-click `start_project.bat`. It starts Django on http://127.0.0.1:8000 and Next.js on http://localhost:3000 and opens the browser. Log in with one of the demo users.

Django also has its own admin panel at http://127.0.0.1:8000/admin/ (user `admin`, password `123456789`). It needs `python manage.py migrate` and `createsuperuser` once on a new database. The normal application does not use it.

**Using the app**

* The top bar has a search box (press `Ctrl+K`) that finds customers, vendors, products, shipments, orders and batches. It only returns what the user is allowed to see.
* **+ New** in the top bar opens the "create" form of any module the user has access to.
* Every list page has its own search box and a refresh button.
* A normal run through: Vendors, Products and Customers (admin), New Shipment, Customs Cleared, Receive, Record payment (admin), New Sales Order, New Transfer, Confirm Receipt, Track Order, Reports.

---

## 7. Tests

`tests\run_tests.bat` runs all the tests on a temporary database named `refrigeration_scms_test`. It creates that database, runs the tests, and deletes it again. Your real data is not touched.

| What | File | Checks |
| :-- | :-- | :-: |
| Database rules (CHECK, UNIQUE, foreign keys) | `tests/db_constraints.sql` | 22 |
| The API: login, access, FIFO, tax, credit limit, shipments, vendor payments, transfers, reports, audit log, users | `tests/api_tests.mjs` | 211 |

To check that the ERD pictures, `schema.sql` and a real database agree (read only):

```
backend\venv\Scripts\python.exe tests\check_erd.py
```

After changing `schema.sql`, run `python docs/erd/build_erd.py` first to redraw the pictures.

---

## 8. Backup and moving to another computer

* `backup_db.bat` saves the whole database into `backups\scms_<date>.dump`.
* `restore_db.bat` puts a backup back (the newest one, or the file you give it). It asks you to type YES first, because it replaces the data that is in the database now.

To take the project to another computer: copy the project folder, run `backup_db.bat` here, copy the `backups` folder too, install PostgreSQL there, create the empty database, and run `restore_db.bat`.

Changing the structure of a database that already holds data is done with the scripts in `backend/migrations/` (they only add things and never delete data), not with `schema.sql`. Take a backup first, then run each file once, in order:

```
cd backend
venv\Scripts\python.exe migrations\apply_migration.py migrations\001_login_access_and_srs_columns.sql
venv\Scripts\python.exe migrations\apply_migration.py migrations\002_vendor_invoice_and_payments.sql
```

A file that was already applied refuses to run a second time. A database made from the current `schema.sql` needs neither of them.

---

## 9. Project folders

```
Refrigeration_SCMS
|-- start_project.bat          starts backend and frontend
|-- backup_db.bat              saves the database
|-- restore_db.bat             restores a saved database
|-- backend
|   |-- schema.sql             creates the 18 tables
|   |-- seed_data.sql          demo data and the five demo users
|   |-- migrations             add-only changes for a database that already has data
|   |-- scms_backend           Django settings and root urls
|   `-- supply_chain
|       |-- models.py          one class per table
|       |-- views.py           the master data, shipment, order, transfer, report and search endpoints
|       |-- account_views.py   login, users, access, audit log endpoints
|       |-- permissions.py     the list of permissions and the default access of each role
|       |-- access.py          checks a permission on every request
|       |-- auth.py            creates and reads the login token
|       |-- audit.py           writes the audit log
|       `-- services
|           |-- fifo_engine.py    FIFO allocation, dock receipt, orders, transfers, tax, credit limit
|           |-- capacity.py       warehouse capacity check
|           `-- audit_service.py  order tracing
|-- frontend/src
|   |-- app                    one folder per page (customers, sales, transfers, users ...)
|   |-- components             menu bar, search box, shared form parts
|   `-- lib                    api calls, login state, types
|-- tests                      database tests, API tests, ERD check, run_tests.bat
`-- docs
    |-- erd                    the ERD pictures and the script that draws them
    |-- architecture.svg       picture used in section 4
    |-- process_flow.svg       picture used in section 2
    |-- ERD_CHANGES.md         what is new compared with the first ERD
    |-- REQUIREMENTS_TRACE.md  SRS and SDS item to table or screen
    `-- VIVA_NOTES.md          study notes and likely questions (Roman Urdu)
```
"# Refrigeration_SCMS" 
