```markdown
# Views & UI (XML)

How to safely extend and modify the Odoo user interface.

## Safely Inheriting a Form View

!!! warning "Warning"
    Always avoid using `position="replace"`. If another module tries to hook into the element you removed, Odoo will crash (Internal Server Error).

Always use `before`, `after`, or hide the element by adding the `d-none` class (or the `invisible` attribute).

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record id="view_order_form_inherit" model="ir.ui.view">
        <field name="name">sale.order.form.inherit.custom</field>
        <field name="model">sale.order</field>
        <field name="inherit_id" ref="sale.view_order_form"/>
        <field name="arch" type="xml">
            
            <!-- 1. Add a field AFTER an existing field -->
            <xpath expr="//field[@name='payment_term_id']" position="after">
                <field name="total_weight"/>
            </xpath>

            <!-- 2. Make a field invisible based on a condition (Odoo 17/18) -->
            <xpath expr="//field[@name='validity_date']" position="attributes">
                <attribute name="invisible">state in ['done', 'cancel']</attribute>
            </xpath>

        </field>
    </record>
</odoo>