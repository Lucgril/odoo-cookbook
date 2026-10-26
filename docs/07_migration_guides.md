# Migration & Version Guide (17 to 20)

<div style="display: flex; gap: 8px; margin-bottom: 20px;">
  <span style="background: #714B67; color: white; padding: 4px 10px; border-radius: 4px; font-size: 0.85em; font-weight: bold;">Odoo 18.0</span>
  <span style="background: #017E84; color: white; padding: 4px 10px; border-radius: 4px; font-size: 0.85em; font-weight: bold;">Odoo 19.0</span>
  <span style="background: #28a745; color: white; padding: 4px 10px; border-radius: 4px; font-size: 0.85em; font-weight: bold;">Odoo 20.0 Ready</span>
</div>

Odoo's framework evolves rapidly. Between Odoo 17 and Odoo 20, several major structural changes were finalized. This guide highlights the key syntactical shifts you must implement to ensure your modules remain compatible with modern Odoo versions.

## 1. The Death of `<tree>` (Replaced by `<list>`)

In earlier versions, tabular views were defined using the `<tree>` tag. Starting heavily in Odoo 17, standardizing in 18, and strictly enforced in 19 and 20, the tag is now officially `<list>`.

**Legacy Syntax (Pre-17):**
```xml
<record id="view_partner_tree" model="ir.ui.view">
    <field name="arch" type="xml">
        <tree string="Partners">
            <field name="name"/>
        </tree>
    </field>
</record>
```

**Modern Syntax (18, 19, 20):**
```xml
<record id="view_partner_list" model="ir.ui.view">
    <!-- Notice the view_mode is 'list', not 'tree' -->
    <field name="arch" type="xml">
        <list string="Partners">
            <field name="name"/>
        </list>
    </field>
</record>
```
*Note: In your window actions (`ir.actions.act_window`), ensure `view_mode` specifies `list,form` instead of `tree,form`.*

## 2. Deprecation of the `attrs` Dictionary

Previously, conditional modifiers (invisible, readonly, required) were packed into a Python-like dictionary evaluated by the client. This is completely deprecated.

**Legacy Syntax (Pre-17):**
```xml
<field name="vat" attrs="{'invisible': [('is_company', '=', False)], 'readonly': [('state', '=', 'done')]}"/>
```

**Modern Syntax (18, 19, 20):**
Attributes are now directly placed on the tag, containing standard Python domain expressions.

```xml
<field name="vat" 
       invisible="not is_company" 
       readonly="state == 'done'"/>
```

## 3. Manifest Versioning Standard

To adhere to OCA (Odoo Community Association) standards, always format the `version` key in your `__manifest__.py` combining the targeted Odoo version and your module's semantic version.

*   For Odoo 18: `'version': '18.0.1.0.0'`
*   For Odoo 19: `'version': '19.0.1.0.0'`
*   For Odoo 20: `'version': '20.0.1.0.0'`

```python
{
    'name': 'My Custom Module',
    'version': '20.0.1.0.0',
    'depends': ['base', 'web'],
    'data': [
        'security/ir.model.access.csv',
        'views/my_model_views.xml',
    ],
    'assets': {
        'web.assets_backend': [
            'my_custom_module/static/src/**/*',
        ],
    },
}
```

## 4. Assets Declaration

The way JavaScript, CSS, and SCSS assets are loaded shifted significantly in Odoo 15+ and remains standard through 20. Never use `<template id="assets_backend" inherit_id="web.assets_backend">` inside XML files anymore.

All assets must be declared inside the `__manifest__.py` under the `'assets'` dictionary using glob patterns (`**/*` is fully supported).