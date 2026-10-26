# ORM & Models

A collection of snippets for model manipulation, computed fields, and Wizards.

## Computed Field with @api.depends

Computed fields are dynamically recalculated when the fields they depend on are modified.

```python
from odoo import models, fields, api

class SaleOrder(models.Model):
    _inherit = 'sale.order'

    total_weight = fields.Float(
        string='Total Weight (kg)', 
        compute='_compute_total_weight',
        store=True # Set to True if you need to group/filter by this field
    )

    @api.depends('order_line.product_id.weight', 'order_line.product_uom_qty')
    def _compute_total_weight(self):
        for order in self:
            weight = 0.0
            for line in order.order_line:
                weight += (line.product_id.weight * line.product_uom_qty)
            order.total_weight = weight