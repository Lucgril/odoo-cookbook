# Actions & Scheduled Tasks (Cron)

This page covers automated processing, asynchronous scheduled cron tasks (`ir.cron`), and contextual Server Actions (`ir.actions.server`) in Odoo 17 and 18.

## 1. Scheduled Actions (`ir.cron`)

Cron jobs run tasks automatically in the background at designated time intervals.

### Python Model Handler (`models/sale_order_automation.py`)

Always structure cron methods as batch handlers with defensive error handling so a single failing record does not abort the entire scheduled job:

```python
import logging
from datetime import timedelta
from odoo import models, fields, api

_logger = logging.getLogger(__name__)

class SaleOrder(models.Model):
    _inherit = 'sale.order'

    @api.model
    def _cron_auto_expire_quotations(self):
        """Scans for overdue quotations and transitions them to 'cancel'."""
        cutoff_date = fields.Date.today() - timedelta(days=30)
        expired_quotes = self.search([
            ('state', 'in', ['draft', 'sent']),
            ('create_date', '<', cutoff_date)
        ])

        _logger.info("Auto-expiring %d quotations older than %s", len(expired_quotes), cutoff_date)
        
        for quote in expired_quotes:
            try:
                quote.action_cancel()
                quote.message_post(
                    body="Quotation automatically cancelled due to 30 days of inactivity.",
                    subtype_xmlid='mail.mt_note'
                )
            except Exception as e:
                _logger.error("Failed to auto-expire quotation ID %s: %s", quote.id, str(e))
```

### Cron XML Definition (`data/cron_data.xml`)

Register the cron in an XML file included under `data` in `__manifest__.py`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <data noupdate="1">
        <record id="ir_cron_auto_expire_quotations" model="ir.cron">
            <field name="name">Sale Order: Auto Expire Inactive Quotations</field>
            <field name="model_id" ref="sale.model_sale_order"/>
            <field name="state">code</field>
            <field name="code">model._cron_auto_expire_quotations()</field>
            <field name="interval_number">1</field>
            <field name="interval_type">days</field>
            <field name="numbercall">-1</field> <!-- -1 indicates indefinite repetitions -->
            <field name="active">True</field>
            <field name="nextcall" eval="(DateTime.now() + relativedelta(days=1)).strftime('%Y-%m-%d 02:00:00')"/>
            <field name="priority">5</field>
        </record>
    </data>
</odoo>
```

!!! warning "noupdate='1'"
    Always wrap cron definitions inside `<data noupdate="1">`. This prevents module updates from overriding custom execution times, toggled active statuses, or interval adjustments made by database administrators in production.

---

## 2. Server Actions (`ir.actions.server`)

Server actions allow running Python code from UI buttons, contextual Action menus (batch actions), or automated triggers.

### Adding a Batch Action to the UI "Actions" Gear

This definition binds a server action directly to the **Actions** menu on list and form views for `sale.order`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record id="action_mark_orders_as_reviewed" model="ir.actions.server">
        <field name="name">Mark as Reviewed</field>
        <field name="model_id" ref="sale.model_sale_order"/>
        <field name="binding_model_id" ref="sale.model_sale_order"/>
        <field name="binding_view_types">list,form</field>
        <field name="state">code</field>
        <field name="code">
# 'records' represents the recordset selected in the view
for record in records:
    if record.state == 'draft':
        record.message_post(body="Batch reviewed by manager.")
        </field>
    </record>
</odoo>
```

### Contextual Variables in Server Action Code

When writing Python code inside `<field name="code">`, the execution context provides several built-in variables:

| Variable                       | Description                                             |
| :----------------------------- | :------------------------------------------------------ |
| `env`                          | The current Odoo `Environment` instance                 |
| `model`                        | The `ir.model` object corresponding to the action       |
| `record`                       | Active single record (in form view)                     |
| `records`                      | Recordset of selected items (in list view)              |
| `action`                       | Action dictionary returned to the web client (optional) |
| `time`, `datetime`, `dateutil` | Standard date/time modules available out of the box     |

---

## 3. Triggering Client Action / Window Action from Python

Return an action dictionary directly from your Python methods to redirect users or display notifications:

```python
def action_view_related_invoices(self):
    self.ensure_one()
    return {
        'name': 'Customer Invoices',
        'type': 'ir.actions.act_window',
        'res_model': 'account.move',
        'view_mode': 'list,form', # In Odoo 18 use 'list,form' (tree is legacy)
        'domain': [('partner_id', '=', self.partner_id.id), ('move_type', '=', 'out_invoice')],
        'context': {'default_partner_id': self.partner_id.id},
    }
```