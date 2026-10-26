# Views & UI (XML)

Extending and customizing Odoo views is one of the most common tasks in module development. Odoo uses an XML-based inheritance engine that resolves view trees in memory during load time without modifying upstream source code.

## 1. XPath Inheritance Rules & Target Positions

When targeting nodes in existing views, prioritize stable technical identifiers (like `name="..."`) over volatile CSS classes or arbitrary tag positions.

| Position     | Effect                                                    | Best For                                                    |
| :----------- | :-------------------------------------------------------- | :---------------------------------------------------------- |
| `after`      | Inserts sibling node immediately following the match      | Adding companion fields or buttons                          |
| `before`     | Inserts sibling node immediately preceding the match      | Adding alerts, prefixes, or primary badges                  |
| `inside`     | Appends child element at the end of the matched container | Adding fields into groups, pages, or headers                |
| `attributes` | Alters attributes on the matched tag in-place             | Setting `invisible`, `readonly`, `required`, or CSS classes |
| `replace`    | Overwrites the matched element and children               | **Discouraged** (breaks dependent child modules)            |

!!! danger "Never Remove Nodes via replace"
    Avoid using `position="replace"` to delete fields. If a third-party or downstream OCA module relies on that field or attempts to inherit from it, Odoo will raise an unresolvable XML view error. Prefer hiding elements via `position="attributes"` with `invisible="1"`.

---

## 2. Safely Inheriting Form Views

Form views host business records. This snippet illustrates common mutations: adding conditional fields, modifying attributes with modern domain syntax, and inserting actions in the status bar.

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record id="view_order_form_inherit_cookbook" model="ir.ui.view">
        <field name="name">sale.order.form.inherit.cookbook</field>
        <field name="model">sale.order</field>
        <field name="inherit_id" ref="sale.view_order_form"/>
        <field name="arch" type="xml">

            <!-- 1. Add action button to header bar -->
            <xpath expr="//header" position="inside">
                <button name="action_open_audit_wizard"
                        string="Launch Audit"
                        type="object"
                        class="btn-secondary"
                        invisible="state not in ['draft', 'sent']"/>
            </xpath>

            <!-- 2. Insert field after a target field -->
            <xpath expr="//field[@name='payment_term_id']" position="after">
                <field name="total_weight" readonly="state == 'cancel'"/>
            </xpath>

            <!-- 3. Make an existing field conditionally invisible (Modern Odoo syntax) -->
            <xpath expr="//field[@name='validity_date']" position="attributes">
                <attribute name="invisible">state in ['done', 'cancel']</attribute>
            </xpath>

            <!-- 4. Add a custom notebook tab -->
            <xpath expr="//notebook" position="inside">
                <page string="Technical Specs" name="tech_specs">
                    <group>
                        <group string="Weights">
                            <field name="total_weight"/>
                        </group>
                    </group>
                </page>
            </xpath>

        </field>
    </record>
</odoo>
```

---

## 3. Extending List / Tree Views with Conditional Decorations

List views display tabular collections. Use `decoration-*` classes (`muted`, `danger`, `success`, `warning`, `info`, `bf` for bold) to highlight rows or columns based on dynamic conditions.

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record id="view_order_tree_inherit_cookbook" model="ir.ui.view">
        <field name="name">sale.order.list.inherit.cookbook</field>
        <field name="model">sale.order</field>
        <field name="inherit_id" ref="sale.view_quotation_tree_with_onboarding"/>
        <field name="arch" type="xml">

            <!-- Apply conditional color styling to the full row -->
            <xpath expr="//list" position="attributes">
                <attribute name="decoration-danger">state == 'cancel'</attribute>
                <attribute name="decoration-success">state == 'sale'</attribute>
            </xpath>

            <!-- Add custom field with optional toggle (can be enabled by user in column selector) -->
            <xpath expr="//field[@name='amount_total']" position="before">
                <field name="total_weight" optional="show" sum="Total Weight"/>
            </xpath>

        </field>
    </record>
</odoo>
```

---

## 4. Extending Search Views (Filters & Group By)

Search views control search bar completions, predefined filters, and group-by clauses.

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record id="view_sales_order_filter_inherit_cookbook" model="ir.ui.view">
        <field name="name">sale.order.search.inherit.cookbook</field>
        <field name="model">sale.order</field>
        <field name="inherit_id" ref="sale.view_sales_order_filter"/>
        <field name="arch" type="xml">

            <!-- 1. Add quick preset filter -->
            <xpath expr="//filter[@name='my_sale_orders_filter']" position="after">
                <filter string="Heavy Shipments (> 50kg)"
                        name="heavy_orders"
                        domain="[('total_weight', '>', 50)]"/>
                <separator/>
            </xpath>

            <!-- 2. Add Group By dimension -->
            <xpath expr="//group" position="inside">
                <filter string="Delivery Date" 
                        name="group_delivery_date" 
                        context="{'group_by': 'commitment_date:month'}"/>
            </xpath>

        </field>
    </record>
</odoo>
```

---

## 5. Kanban View Snippet

Kanban views render card-based visual boards. This snippet illustrates card structure, color pickers, and user avatars.

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record id="view_custom_task_kanban" model="ir.ui.view">
        <field name="name">custom.task.kanban</field>
        <field name="model">custom.task</field>
        <field name="arch" type="xml">
            <kanban default_group_by="state" sample="1">
                <field name="id"/>
                <field name="name"/>
                <field name="user_id"/>
                <field name="state"/>
                <templates>
                    <t t-name="card">
                        <div class="d-flex flex-column">
                            <div class="d-flex justify-content-between align-items-center mb-1">
                                <strong class="fs-6"><field name="name"/></strong>
                            </div>
                            <div class="text-muted small">
                                <field name="user_id" widget="many2one_avatar_user"/>
                            </div>
                            <div class="mt-2 d-flex justify-content-end">
                                <field name="state" widget="label_selection"
                                       options="{'classes': {'draft': 'default', 'done': 'success'}}"/>
                            </div>
                        </div>
                    </t>
                </templates>
            </kanban>
        </field>
    </record>
</odoo>
```

---

## 6. Triggering a Wizard Action via XML Button

To invoke a popup wizard directly from a view, trigger an `ir.actions.act_window` record and pass the source record ID in the context:

```xml
<!-- Button in parent form view -->
<button name="%(action_open_mass_update_wizard)d"
        type="action"
        string="Mass Update"
        class="oe_highlight"
        context="{'default_target_model': active_model}"/>

<!-- Window action definition -->
<record id="action_open_mass_update_wizard" model="ir.actions.act_window">
    <field name="name">Status Mass Update</field>
    <field name="res_model">mass.update.wizard</field>
    <field name="view_mode">form</field>
    <field name="target">new</field> <!-- 'new' opens modal popup dialog -->
    <field name="binding_model_id" ref="sale.model_sale_order"/>
    <field name="binding_view_types">list</field> <!-- Appears in Action dropdown in list view -->
</record>
```