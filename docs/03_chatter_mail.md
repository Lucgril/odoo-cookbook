```markdown
# Chatter & Mail

The `mail.thread` module is fundamental in Odoo for activity logging.

## Logging a Note and Sending an Email via Python

This snippet is useful when an automated action (or a button) needs to send a communication without opening the standard email composition window for the user.

```python
from odoo import models

class SaleOrder(models.Model):
    _inherit = 'sale.order'

    def action_custom_notification(self):
        for record in self:
            # 1. Post a silent internal note in the chatter (Yellow background)
            record.message_post(
                body="Automated processing completed successfully.",
                message_type='comment',
                subtype_xmlid='mail.mt_note'
            )

            # 2. Send an email template bypassing the queue (instant send)
            template = self.env.ref('sale.email_template_edi_sale', raise_if_not_found=False)
            if template:
                template.send_mail(
                    record.id, 
                    force_send=True, 
                    email_values={'email_to': 'admin@company.com'}
                )