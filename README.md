# weary-spider
from flask import Flask, render_template, request, redirect, url_for, flash
import sqlite3
from datetime import date, timedelta
from pathlib import Path

app = Flask(__name__)
app.secret_key = "change-this-secret-key"
DB = Path(__file__).with_name("subscriptions.db")

def db():
    conn = sqlite3.connect(DB)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA foreign_keys = ON")
    return conn

def init_db():
    with db() as con:
        con.executescript("""
        CREATE TABLE IF NOT EXISTS customers (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL, email TEXT NOT NULL UNIQUE, phone TEXT DEFAULT ''
        );
        CREATE TABLE IF NOT EXISTS plans (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL, price REAL NOT NULL CHECK(price >= 0),
            billing_cycle TEXT NOT NULL, description TEXT DEFAULT ''
        );
        CREATE TABLE IF NOT EXISTS subscriptions (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            customer_id INTEGER NOT NULL REFERENCES customers(id),
            plan_id INTEGER NOT NULL REFERENCES plans(id),
            start_date TEXT NOT NULL, next_billing TEXT NOT NULL,
            status TEXT NOT NULL DEFAULT 'Active'
        );
        CREATE TABLE IF NOT EXISTS invoices (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            subscription_id INTEGER NOT NULL REFERENCES subscriptions(id),
            customer_id INTEGER NOT NULL REFERENCES customers(id),
            amount REAL NOT NULL, issue_date TEXT NOT NULL,
            due_date TEXT NOT NULL, status TEXT NOT NULL DEFAULT 'Unpaid'
        );
        CREATE TABLE IF NOT EXISTS payments (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            invoice_id INTEGER NOT NULL REFERENCES invoices(id),
            amount REAL NOT NULL, method TEXT NOT NULL,
            paid_date TEXT NOT NULL, note TEXT DEFAULT ''
        );
        """)
        if con.execute("SELECT COUNT(*) FROM customers").fetchone()[0] == 0:
            con.execute("INSERT INTO customers(name,email,phone) VALUES(?,?,?)",
                        ("Demo Customer", "demo@example.com", "9876543210"))
            con.execute("INSERT INTO plans(name,price,billing_cycle,description) VALUES(?,?,?,?)",
                        ("Starter", 499, "Monthly", "Basic subscription plan"))
            cid = con.execute("SELECT id FROM customers LIMIT 1").fetchone()["id"]
            pid = con.execute("SELECT id FROM plans LIMIT 1").fetchone()["id"]
            today = date.today()
            next_date = today + timedelta(days=30)
            cur = con.execute("""INSERT INTO subscriptions(customer_id,plan_id,start_date,next_billing,status)
                               VALUES(?,?,?,?,?)""",
                              (cid, pid, today.isoformat(), next_date.isoformat(), "Active"))
            con.execute("""INSERT INTO invoices(subscription_id,customer_id,amount,issue_date,due_date,status)
                           VALUES(?,?,?,?,?,?)""",
                        (cur.lastrowid, cid, 499, today.isoformat(),
                         (today + timedelta(days=7)).isoformat(), "Unpaid"))

@app.route("/")
def index():
    with db() as con:
        customers = con.execute("SELECT * FROM customers ORDER BY id DESC").fetchall()
        plans = con.execute("SELECT * FROM plans ORDER BY id DESC").fetchall()
        subscriptions = con.execute("""
            SELECT s.*, c.name customer, p.name plan, p.price
            FROM subscriptions s JOIN customers c ON c.id=s.customer_id
            JOIN plans p ON p.id=s.plan_id ORDER BY s.id DESC
        """).fetchall()
        invoices = con.execute("""
            SELECT i.*, c.name customer FROM invoices i
            JOIN customers c ON c.id=i.customer_id ORDER BY i.id DESC
        """).fetchall()
        payments = con.execute("""
            SELECT pay.*, i.id invoice_number, c.name customer
            FROM payments pay JOIN invoices i ON i.id=pay.invoice_id
            JOIN customers c ON c.id=i.customer_id ORDER BY pay.id DESC
        """).fetchall()
        active = con.execute("SELECT COUNT(*) FROM subscriptions WHERE status='Active'").fetchone()[0]
        revenue = con.execute("SELECT COALESCE(SUM(amount),0) FROM payments").fetchone()[0]
        outstanding = con.execute("SELECT COALESCE(SUM(amount),0) FROM invoices WHERE status!='Paid'").fetchone()[0]
        overdue = con.execute("SELECT COUNT(*) FROM invoices WHERE status!='Paid' AND due_date < ?",
                              (date.today().isoformat(),)).fetchone()[0]
    return render_template("index.html", customers=customers, plans=plans,
        subscriptions=subscriptions, invoices=invoices, payments=payments,
        active=active, revenue=revenue, outstanding=outstanding, overdue=overdue,
        today=date.today().isoformat())

@app.post("/customers")
def add_customer():
    try:
        with db() as con:
            con.execute("INSERT INTO customers(name,email,phone) VALUES(?,?,?)",
                (request.form["name"].strip(), request.form["email"].strip().lower(),
                 request.form.get("phone","").strip()))
        flash("Customer added successfully.", "success")
    except sqlite3.IntegrityError:
        flash("That email address already exists.", "error")
    return redirect(url_for("index") + "#customers")

@app.post("/plans")
def add_plan():
    try:
        price = float(request.form["price"])
        if price < 0: raise ValueError()
        with db() as con:
            con.execute("INSERT INTO plans(name,price,billing_cycle,description) VALUES(?,?,?,?)",
                (request.form["name"].strip(), price, request.form["billing_cycle"],
                 request.form.get("description","").strip()))
        flash("Plan added successfully.", "success")
    except (ValueError, sqlite3.Error):
        flash("Please enter valid plan details.", "error")
    return redirect(url_for("index") + "#plans")

@app.post("/subscriptions")
def add_subscription():
    try:
        start = date.fromisoformat(request.form["start_date"])
        customer_id, plan_id = int(request.form["customer_id"]), int(request.form["plan_id"])
        with db() as con:
            plan = con.execute("SELECT * FROM plans WHERE id=?", (plan_id,)).fetchone()
            customer = con.execute("SELECT * FROM customers WHERE id=?", (customer_id,)).fetchone()
            if not plan or not customer: raise ValueError()
            days = {"Daily":1, "Weekly":7, "Monthly":30, "Yearly":365}[plan["billing_cycle"]]
            next_date = start + timedelta(days=days)
            cur = con.execute("""INSERT INTO subscriptions(customer_id,plan_id,start_date,next_billing,status)
                VALUES(?,?,?,?, 'Active')""", (customer_id, plan_id, start.isoformat(), next_date.isoformat()))
            con.execute("""INSERT INTO invoices(subscription_id,customer_id,amount,issue_date,due_date,status)
                VALUES(?,?,?,?,?,'Unpaid')""",
                (cur.lastrowid, customer_id, plan["price"], start.isoformat(),
                 (start + timedelta(days=7)).isoformat()))
        flash("Subscription created and its first invoice generated.", "success")
    except (ValueError, KeyError, TypeError):
        flash("Please select a valid customer, plan and start date.", "error")
    return redirect(url_for("index") + "#subscriptions")

@app.post("/subscriptions/<int:item_id>/status")
def update_subscription(item_id):
    status = request.form.get("status")
    if status not in ("Active", "Paused", "Closed"):
        flash("Invalid subscription status.", "error")
    else:
        with db() as con:
            con.execute("UPDATE subscriptions SET status=? WHERE id=?", (status, item_id))
        flash("Subscription status updated.", "success")
    return redirect(url_for("index") + "#subscriptions")

@app.post("/invoices/<int:item_id>/status")
def update_invoice(item_id):
    status = request.form.get("status")
    if status not in ("Unpaid", "Paid", "Cancelled"):
        flash("Invalid invoice status.", "error")
    else:
        with db() as con:
            con.execute("UPDATE invoices SET status=? WHERE id=?", (status, item_id))
        flash("Invoice status updated.", "success")
    return redirect(url_for("index") + "#invoices")

@app.post("/payments")
def add_payment():
    try:
        invoice_id = int(request.form["invoice_id"])
        amount = float(request.form["amount"])
        if amount <= 0: raise ValueError()
        with db() as con:
            invoice = con.execute("SELECT * FROM invoices WHERE id=?", (invoice_id,)).fetchone()
            if not invoice or invoice["status"] == "Cancelled": raise ValueError()
            con.execute("INSERT INTO payments(invoice_id,amount,method,paid_date,note) VALUES(?,?,?,?,?)",
                (invoice_id, amount, request.form["method"], request.form["paid_date"],
                 request.form.get("note","").strip()))
            paid = con.execute("SELECT COALESCE(SUM(amount),0) FROM payments WHERE invoice_id=?",
                               (invoice_id,)).fetchone()[0]
            if paid >= invoice["amount"]:
                con.execute("UPDATE invoices SET status='Paid' WHERE id=?", (invoice_id,))
        flash("Payment recorded. Invoice marked paid if the full amount was received.", "success")
    except (ValueError, KeyError):
        flash("Please enter valid payment details.", "error")
    return redirect(url_for("index") + "#payments")

if __name__ == "__main__":
    init_db()
    app.run(debug=True)
