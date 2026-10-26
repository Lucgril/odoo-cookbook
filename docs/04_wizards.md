# Wizards (Transient Models)

Wizards in Odoo handle interactive, multi-step, or confirmation workflows. They are powered by `models.TransientModel`, meaning their records are temporary and automatically pruned from the database by an internal vacuum cron job.

## 1. Core Architecture of a Wizard

A complete wizard requires three elements:
1. A Python model inheriting from `models.TransientModel`.
2. A dedicated Form view with a `<footer/>` container containing action and cancellation buttons.
3. An action record (`ir.actions.act_window`) configured with `target="new"`.

### Python Model (`models/sale_order_cancel_wizard.py`)

```python
from odoo import models, fields, api
from odoo.exceptions import UserError

class SaleOrderCancelWizard(models.TransientModel):
    _name = 'sale.order.cancel.wizard'
    _description = 'Sale Order Cancellation Wizard'

    order_id = fields.Many2one(
        'sale.order',
        string="Order",
        required=True,
        default=lambda self: self.env.context.get('active_id')
    )
    reason = fields.Selection([
        ('out_of_stock', 'Product Out of Stock'),
        ('customer_request', 'Customer Requested Cancellation'),
        ('price_issue', 'Pricing Discrepancy'),
        ('other', 'Other Reason'),
    ], string="Reason", required=True, default='customer_request')

    additional_notes = fields.Text(string="Additional Notes")

    def action_confirm_cancellation(self):
        """Executes the cancellation workflow and logs context into the chatter."""
        self.ensure_one()
        if not self.order_id:
            raise UserError("No linked order found in context.")

        # Post cancellation reason in order chatter
        note_body = (
            f"<b>Order Cancelled via Wizard</b><br/>"
            f"Reason: {dict(self._fields['reason'].selection).get(self.reason)}<br/>"
            f"Notes: {self.additional_notes or 'N/A'}"
        )
        self.order_id.message_post(body=note_body, subtype_xmlid='mail.mt_note')

        # Execute order cancellation
        self.order_id.action_cancel()

        # Close the modal dialog
        return {'type': 'ir.actions.act_window_close'}
```

### XML Form View & Action (`views/sale_order_cancel_wizard_views.xml`)

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record id="view_sale_order_cancel_wizard_form" model="ir.ui.view">
        <field name="name">sale.order.cancel.wizard.form</field>
        <field name="model">sale.order.cancel.wizard</field>
        <field name="arch" type="xml">
            <form string="Cancel Order">
                <sheet>
                    <group>
                        <field name="order_id" readonly="1"/>
                        <field name="reason"/>
                        <field name="additional_notes" 
                               invisible="reason != 'other'" 
                               required="reason == 'other'"/>
                    </group>
                </sheet>
                <footer>
                    <button name="action_confirm_cancellation" 
                            string="Confirm Cancel" 
                            type="object" 
                            class="btn-danger"/>
                    <button string="Discard" 
                            class="btn-secondary" 
                            special="cancel"/>
                </footer>
            </form>
        </field>
    </record>

    <!-- Action that triggers the modal popup -->
    <record id="action_sale_order_cancel_wizard" model="ir.actions.act_window">
        <field name="name">Cancel Order Confirmation</field>
        <field name="res_model">sale.order.cancel.wizard</field>
        <field name="view_mode">form</field>
        <field name="target">new</field> <!-- Mandatory: renders modal popup -->
    </record>
</odoo>
```

---

## 2. Multi-Step Wizard (Step-by-Step Flow)

You can build sequential wizards (e.g., Step 1: Options -> Step 2: Confirmation) by managing a `state` field on the transient record and returning an action reloading the same wizard ID.

```python
from odoo import models, fields

class MultiStepExportWizard(models.TransientModel):
    _name = 'multi.step.export.wizard'
    _description = 'Multi-Step Export Wizard'

    state = fields.Selection([
        ('step1', 'Step 1: Configuration'),
        ('step2', 'Step 2: Summary & Run'),
    ], default='step1')

    export_format = fields.Selection([
        ('csv', 'CSV File'),
        ('xlsx', 'Excel Sheet'),
    ], string="File Format", default='xlsx')
    
    include_archived = fields.Boolean(string="Include Archived Records")

    def action_go_to_step2(self):
        self.ensure_one()
        self.state = 'step2'
        # Reopen current wizard dialog preserving in-memory state
        return {
            'type': 'ir.actions.act_window',
            'res_model': self._name,
            'res_id': self.id,
            'view_mode': 'form',
            'target': 'new',
        }

    def action_go_to_step1(self):
        self.ensure_one()
        self.state = 'step1'
        return {
            'type': 'ir.actions.act_window',
            'res_model': self._name,
            'res_id': self.id,
            'view_mode': 'form',
            'target': 'new',
        }

    def action_execute(self):
        self.ensure_one()
        # Launch actual business routine here
        return {'type': 'ir.actions.act_window_close'}
```

Corresponding XML buttons in `<footer>`:
```xml
<footer>
    <!-- Step 1 Controls -->
    <button name="action_go_to_step2" string="Next" type="object" class="btn-primary" invisible="state != 'step1'"/>

    <!-- Step 2 Controls -->
    <button name="action_go_to_step1" string="Back" type="object" class="btn-secondary" invisible="state != 'step2'"/>
    <button name="action_execute" string="Start Processing" type="object" class="btn-primary" invisible="state != 'step2'"/>

    <button string="Cancel" class="btn-secondary" special="cancel"/>
</footer>
```