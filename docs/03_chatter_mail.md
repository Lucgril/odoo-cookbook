# Chatter & Mail

The OpenERP/Odoo communication engine relies heavily on the `mail.thread` and `mail.activity.mixin` mixins. Integrating Chatter into custom business objects provides an audit trail, follower notifications, scheduled activities, and direct messaging out of the box.

## Core Prerequisites

To leverage Chatter features on a custom model, ensure both the model definition and view definitions declare the proper dependencies:

### 1. Model Mixins
Your model must inherit from `mail.thread` (for messages/notes) and optionally `mail.activity.mixin` (for scheduled tasks):

```python
from odoo import models, fields

class CustomProject(models.Model):
    _name = 'custom.project'
    _description = 'Custom Project'
    _inherit = ['mail.thread', 'mail.activity.mixin']

    name = fields.Char(string="Title", required=True, tracking=True)
    state = fields.Selection([
        ('draft', 'Draft'),
        ('in_progress', 'In Progress'),
        ('done', 'Done'),
    ], default='draft', tracking=True)
```

!!! tip "Field Tracking"
    Adding `tracking=True` to fields automatically records all value changes inside the Chatter feed without writing custom Python hooks.

### 2. View Definition
Append the Chatter widget below the `<sheet>` tag in your form view:

```xml
<sheet>
    <!-- Your form content -->
</sheet>
<chatter/>
```

---

## Message Posting Mechanics

The `message_post()` method handles all programmatic communication in Odoo. Key subtypes dictate audience visibility:

| Subtype XMLID     | Visual Style             | Intended Audience           | Notifications Sent?    |
| :---------------- | :----------------------- | :-------------------------- | :--------------------- |
| `mail.mt_note`    | Light grey / yellow tint | Internal users only         | No (silent log)        |
| `mail.mt_comment` | Standard bubble          | Record followers & partners | Yes (dispatches email) |

---

## Logging a Note and Sending an Email via Python

This snippet demonstrates silent internal auditing alongside targeted transactional email delivery bypassing background queues:

```python
from odoo import models

class SaleOrder(models.Model):
    _inherit = 'sale.order'

    def action_custom_notification(self):
        for record in self:
            # 1. Post a silent internal note to record history without alerting followers
            record.message_post(
                body="Automated order verification completed successfully.",
                message_type='comment',
                subtype_xmlid='mail.mt_note'
            )

            # 2. Dispatch an email using a pre-configured template (immediate delivery)
            template = self.env.ref('sale.email_template_edi_sale', raise_if_not_found=False)
            if template:
                template.send_mail(
                    record.id,
                    force_send=True,
                    email_values={'email_to': 'audit@example.com'}
                )

    def action_post_message_with_attachment(self, file_content_base64, filename):
        """Attaches dynamic binary data directly into the chatter timeline."""
        for record in self:
            record.message_post(
                body="System report generated and attached.",
                subtype_xmlid='mail.mt_note',
                attachments=[(filename, file_content_base64)]
            )
```