# ORM & Models

A comprehensive reference for working with the Odoo Object-Relational Mapping (ORM) layer, business logic hooks, validations, and transient interactions.

---

## 1. Computed Fields (`@api.depends`)

Computed fields calculate their values on the fly or persist them using `store=True`. Always decorate compute methods with `@api.depends` so Odoo knows when to recalculate them.

```python
from odoo import models, fields, api

class SaleOrder(models.Model):
    _inherit = 'sale.order'

    total_weight = fields.Float(
        string='Total Weight (kg)',
        compute='_compute_total_weight',
        store=True,  # Set to True to allow sorting and filtering in list/search views
        help="Aggregated weight of all order lines."
    )

    @api.depends('order_line.product_id.weight', 'order_line.product_uom_qty')
    def _compute_total_weight(self):
        for order in self:
            weight = 0.0
            for line in order.order_line:
                weight += (line.product_id.weight or 0.0) * line.product_uom_qty
            order.total_weight = weight
```

!!! tip "Inverse & Search on Compute Fields"
    If you need users to manually edit a computed field, define an `inverse='_inverse_method'`. If `store=False` and you need to filter on it, define a `search='_search_method'`.

---

## 2. Business Validation Constraints (`@api.constrains`)

Use `@api.constrains` for complex cross-field validations that cannot be handled by simple database constraints (`_sql_constraints`).

```python
from odoo import models, fields, api
from odoo.exceptions import ValidationError

class SaleOrder(models.Model):
    _inherit = 'sale.order'

    expected_delivery = fields.Date(string="Expected Delivery Date")

    @api.constrains('expected_delivery', 'date_order')
    def _check_expected_delivery(self):
        for record in self:
            if record.expected_delivery and record.date_order:
                if record.expected_delivery < record.date_order.date():
                    raise ValidationError(
                        "The expected delivery date cannot precede the order confirmation date."
                    )
```

---

## 3. Overriding CRUD Methods (`create` & `write`)

When intercepting record creation, always implement `@api.model_create_multi` to support batch imports and high-performance multi-record creation.

```python
from odoo import models, fields, api

class CustomTask(models.Model):
    _name = 'custom.task'
    _description = 'Custom Task with Sequence'

    name = fields.Char(
        string='Reference',
        required=True,
        copy=False,
        readonly=True,
        default='New'
    )
    is_urgent = fields.Boolean(default=False)

    @api.model_create_multi
    def create(self, vals_list):
        for vals in vals_list:
            if vals.get('name', 'New') == 'New':
                # Fetch reference from ir.sequence
                vals['name'] = (
                    self.env['ir.sequence'].next_by_code('custom.task.code') or 'New'
                )
        return super().create(vals_list)

    def write(self, vals):
        # Example: pre-write hook
        if 'is_urgent' in vals and vals['is_urgent']:
            # Perform custom pre-update validation or side effects
            pass
        return super().write(vals)
```

---

## 4. Recordset Operations (`filtered`, `mapped`, `sorted`)

Avoid running nested Python loops over database records. Use Odoo's native recordset utilities to maintain fast, readable, in-memory operations.

```python
# 1. Search with standard domain
confirmed_orders = self.env['sale.order'].search([
    ('state', '=', 'sale'),
    ('partner_id.country_id.code', '=', 'IT'),
])

# 2. Filter records in memory using a lambda or field name
high_value_orders = confirmed_orders.filtered(lambda order: order.amount_total >= 5000)

# 3. Extract attributes or linked records into lists / recordsets
customer_emails = confirmed_orders.mapped('partner_id.email')  # Python list of unique emails
all_order_lines = confirmed_orders.mapped('order_line')         # Flat recordset of sale.order.line

# 4. Sort recordsets in memory
sorted_orders = confirmed_orders.sorted(key=lambda o: o.date_order, reverse=True)
```

---

## 5. Modifying Environment & Context (`with_context`)

The context carries session metadata (active language, default values, user timezone). You can pass keys to alter behavior or silence background jobs during heavy batch operations.

```python
# Silence chatter notifications and tracking during bulk migrations
bulk_orders = self.env['sale.order'].browse([10, 11, 12])

bulk_orders.with_context(
    tracking_disable=True,
    mail_auto_subscribe_no_notify=True,
    mail_create_nolog=True
).write({'state': 'done'})

# Read translated field values in a specific language
french_name = self.env['product.product'].with_context(lang='fr_FR').browse(1).name
```