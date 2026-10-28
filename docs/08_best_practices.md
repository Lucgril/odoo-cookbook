# Best Practices & Common Pitfalls

<div style="display: flex; gap: 8px; margin-bottom: 20px;">
  <span style="background: #714B67; color: white; padding: 4px 10px; border-radius: 4px; font-size: 0.85em; font-weight: bold;">Odoo 18.0</span>
  <span style="background: #017E84; color: white; padding: 4px 10px; border-radius: 4px; font-size: 0.85em; font-weight: bold;">Odoo 19.0</span>
  <span style="background: #28a745; color: white; padding: 4px 10px; border-radius: 4px; font-size: 0.85em; font-weight: bold;">Odoo 20.0 Ready</span>
</div>

Developing in Odoo can sometimes be tricky. Below is a collection of the most common pitfalls, hidden traps, and architectural best practices to keep your code fast, safe, and bug-free.

---

### 1. The "Expected Singleton" Error
This is the most common Odoo error. It happens when you try to access a field or call a method on a recordset containing multiple records, but the code expects only one.

**Wrong Way:**
```python
def action_do_something(self):
    # If 'self' contains 3 records, this crashes with ValueError: Expected singleton
    if self.state == 'draft':
        self.state = 'done'
```

**Correct Way (Looping or ensure_one):**
```python
def action_do_something(self):
    # Option A: Handle multiple records gracefully
    for record in self:
        if record.state == 'draft':
            record.state = 'done'

def action_print_report(self):
    # Option B: Force a singleton (crashes early if multiple records are passed)
    self.ensure_one() 
    return self.env.ref('my_module.my_report_action').report_action(self)
```

### 2. The N+1 Query Problem (Performance)
Writing or searching inside a `for` loop generates massive amounts of SQL queries, slowing down the system significantly. Always use batch operations.

**Wrong Way (Very Slow):**
```python
for order in self:
    # Executes a SQL query for EVERY single order in the loop
    lines = self.env['sale.order.line'].search([('order_id', '=', order.id)])
    order.write({'is_processed': True}) # Executes an UPDATE query per order
```

**Correct Way (Fast & Optimized):**
```python
# One single SQL query to fetch all lines for all orders
all_lines = self.env['sale.order.line'].search([('order_id', 'in', self.ids)])

# One single SQL query to update all orders at once
self.write({'is_processed': True})
```

### 3. Elevating Privileges and Multi-Company Context
Sometimes a user needs to trigger an action (like confirming an order) that creates records in another model where they don't have write access, or in a specific company environment.

```python
def action_advanced_process(self):
    for record in self:
        # 1. sudo() bypasses record rules and access rights (use with caution)
        # 2. with_company() ensures accounting entries are created in the correct company
        record.env['account.move'].sudo().with_company(record.company_id).create({
            'move_type': 'out_invoice',
            'partner_id': record.partner_id.id,
            'company_id': record.company_id.id,
        })
```

### 4. Preventing Deletion of Processed Records
By default, users with delete access can remove records anytime. It is a best practice to override the `unlink` method to prevent deleting records that are already confirmed.

```python
from odoo.exceptions import UserError

class SaleOrder(models.Model):
    _inherit = 'sale.order'

    def unlink(self):
        for record in self:
            if record.state not in ['draft', 'cancel']:
                raise UserError("You can only delete draft or cancelled orders.")
        # Call super to perform the actual deletion
        return super().unlink()
```

### 5. The "NewId" Trap in Computes and Onchanges
When Odoo processes `@api.depends` or `@api.onchange` on a form before it is saved, the record doesn't exist in the database yet. Its ID is a virtual `NewId`. If you try to pass this ID to an external API, a raw SQL query, or another environment, it will crash.

**Problem (Crashes on unsaved records):**
```python
@api.depends('amount')
def _compute_external_sync(self):
    for record in self:
        # If 'record' is not saved yet, record.id is an object like <NewId 0x...>
        self.env.cr.execute("SELECT * FROM some_table WHERE order_id = %s", [record.id])
```

**Correct Way (Using `_origin`):**
```python
@api.depends('amount')
def _compute_external_sync(self):
    for record in self:
        # _origin returns the actual database record (if it exists) or an empty recordset
        real_id = record._origin.id
        if real_id:
            self.env.cr.execute("SELECT * FROM some_table WHERE order_id = %s", [real_id])
```

### 6. Broken Transactions (Savepoints)
If you try to catch an exception (e.g., a constraint violation or a database lock) using a `try/except` block, Odoo's database cursor will still be marked as "failed". Any subsequent ORM operation in the same transaction will crash with a `psycopg2` InternalError.

**Wrong Way (Transaction gets poisoned):**
```python
def action_sync_all(self):
    for record in self:
        try:
            record.action_risky_sync() # If this raises a DB error...
        except Exception:
            # ...the cursor is broken. The next loop iteration will crash immediately!
            continue 
```

**Correct Way (Using Savepoints):**
```python
def action_sync_all(self):
    for record in self:
        try:
            # Create a savepoint before the risky operation
            with self.env.cr.savepoint():
                record.action_risky_sync()
        except Exception as e:
            # If it fails, only this specific savepoint is rolled back.
            # The transaction stays healthy for the next records.
            record.message_post(body=f"Sync failed: {e}")
```

### 7. Context Pollution (Default values leaking)
When you pass the `context` to a new window action or a method, it often contains keys like `default_name` or `default_partner_id`. If you use this context to create other unrelated records (e.g., logging a history line), Odoo will try to apply those `default_*` keys to the new model, often causing crashes if the fields don't exist.

**Wrong Way:**
```python
def create_invoice(self):
    # If self.env.context contains 'default_type': 'sale', it leaks into the wizard!
    return self.env['some.wizard'].create({
        'order_id': self.id
    })
```

**Correct Way (Clean the context):**
```python
def create_invoice(self):
    # Remove specific dangerous keys, or pass a completely clean context
    clean_context = dict(self.env.context)
    clean_context.pop('default_type', None)
    
    return self.env['some.wizard'].with_context(clean_context).create({
        'order_id': self.id
    })
```

### 8. Raw SQL Bypasses the ORM Cache
Sometimes you need to use `self.env.cr.execute()` for massive performance gains. However, raw SQL entirely bypasses the Odoo ORM. This means the ORM cache is not updated, and computed fields depending on your changes will not trigger.

**Wrong Way (Cache is out of sync):**
```python
def action_mass_update(self):
    # Updates the database directly
    self.env.cr.execute("UPDATE sale_order SET state = 'done' WHERE id IN %s", [tuple(self.ids)])
    
    for order in self:
        # WRONG: The ORM cache still thinks the state is whatever it was before the SQL query!
        print(order.state) 
```

**Correct Way (Invalidate the cache):**
```python
def action_mass_update(self):
    self.env.cr.execute("UPDATE sale_order SET state = 'done' WHERE id IN %s", [tuple(self.ids)])
    
    # Clear the cache for these specific records so the ORM fetches fresh data
    self.invalidate_recordset(['state']) 
    
    # Or, if you updated multiple tables: self.env.invalidate_all()
```

### 9. The Timezone Illusion (UTC vs Local Time)
In Odoo, `fields.Datetime` is **always stored in UTC** in the PostgreSQL database. It is only converted to the user's local timezone visually in the frontend. If you do date math in Python or format a datetime to a string without converting it first, your times will be off by several hours.

**Wrong Way (Printing raw UTC):**
```python
def print_time(self):
    # If the user is in Rome (UTC+2) and creates a record at 14:00, 
    # the DB stores 12:00. This will print 12:00!
    return f"Created at: {self.create_date.strftime('%H:%M')}"
```

**Correct Way (Converting to user's timezone):**
```python
import pytz
from odoo.tools import format_datetime

def print_time(self):
    # Option A: Use Odoo's built-in formatting tool (handles tz and lang automatically)
    local_time_str = format_datetime(self.env, self.create_date, dt_format='HH:mm')
    
    # Option B: Manual timezone conversion using pytz
    user_tz = pytz.timezone(self.env.user.tz or 'UTC')
    local_dt = self.create_date.replace(tzinfo=pytz.utc).astimezone(user_tz)
```

### 10. Modifying One2many/Many2many with `Command`
In older Odoo versions, you had to use weird tuple structures like `(0, 0, {})` or `(4, id)` to modify relation fields. Since Odoo 16+, and heavily enforced in Odoo 18-20, you should use the `Command` namespace. It prevents syntax errors and makes your code self-documenting.

**Wrong Way (Old Tuple Syntax):**
```python
def add_lines(self):
    self.write({
        'line_ids': [
            (0, 0, {'product_id': 1, 'price': 10}), # Create
            (4, 5, False),                          # Link existing ID 5
            (5, 0, 0)                               # Unlink all
        ]
    })
```

**Correct Way (Modern `Command` Class):**
```python
from odoo import Command

def add_lines(self):
    self.write({
        'line_ids': [
            Command.create({'product_id': 1, 'price': 10}),
            Command.link(5),
            Command.clear()
        ]
    })
```