# OWL & Frontend JavaScript

<div style="display: flex; gap: 8px; margin-bottom: 20px;">
  <span style="background: #714B67; color: white; padding: 4px 10px; border-radius: 4px; font-size: 0.85em; font-weight: bold;">Odoo 18.0</span>
  <span style="background: #017E84; color: white; padding: 4px 10px; border-radius: 4px; font-size: 0.85em; font-weight: bold;">Odoo 19.0</span>
  <span style="background: #28a745; color: white; padding: 4px 10px; border-radius: 4px; font-size: 0.85em; font-weight: bold;">Odoo 20.0 Ready</span>
</div>

Odoo's frontend is entirely powered by the **Odoo Web Library (OWL)**. Starting from Odoo 16, and solidifying through Odoo 18, 19, and 20, legacy widgets have been entirely replaced by OWL components, hooks, and registries.

## 1. Creating a Custom Field Component

To create a new field type (e.g., a custom text input that converts text to uppercase), you need a JavaScript component and to register it in the `fields` registry.

```javascript
/** @odoo-module **/

import { Component, xml } from "@odoo/owl";
import { registry } from "@web/core/registry";
import { standardFieldProps } from "@web/views/fields/standard_field_props";
import { useService } from "@web/core/utils/hooks";

export class UppercaseCharField extends Component {
    static template = xml`
        <div class="o_uppercase_field">
            <input 
                type="text" 
                class="o_input"
                t-att-value="props.record.data[props.name]"
                t-on-change="onChange"
            />
        </div>
    `;
    
    static props = {
        ...standardFieldProps,
    };

    setup() {
        this.notificationService = useService("notification");
    }

    onChange(ev) {
        const newValue = ev.target.value.toUpperCase();
        
        // Update the record's value in the model
        this.props.record.update({ [this.props.name]: newValue });
        
        // Use a standard service (notification)
        this.notificationService.add("Value converted to uppercase", {
            type: "success",
        });
    }
}

// Register the component so it can be used in XML views as widget="uppercase_char"
registry.category("fields").add("uppercase_char", {
    component: UppercaseCharField,
});
```

To use this in your XML form view:
```xml
<field name="my_char_field" widget="uppercase_char"/>
```

## 2. Patching an Existing Component

Often, you want to extend an existing component (like the FormController or a standard widget) to add a lifecycle hook or intercept a method. In modern Odoo, we use the `patch` utility.

```javascript
/** @odoo-module **/

import { FormController } from "@web/views/form/form_controller";
import { patch } from "@web/core/utils/patch";
import { onWillStart } from "@odoo/owl";

patch(FormController.prototype, {
    
    setup() {
        super.setup(...arguments);
        
        onWillStart(async () => {
            console.log("Form view is about to start loading...");
        });
    },

    async saveButtonClicked() {
        // Intercept the save action
        console.log("Intercepted save button click!");
        
        // Always call the super method to maintain standard behavior
        return await super.saveButtonClicked(...arguments);
    }
});
```

!!! tip "Patching best practices"
    Always call `super.methodName(...arguments)` when overriding existing functions to avoid breaking the core framework logic. Do not use arrow functions if you need access to `this` via `super`.

## 3. Utilizing Standard Services (RPC, ORM, Notification)

Odoo provides several services that can be injected into any OWL component using the `useService` hook.

```javascript
/** @odoo-module **/

import { Component } from "@odoo/owl";
import { useService } from "@web/core/utils/hooks";

export class MyCustomAction extends Component {
    setup() {
        this.rpc = useService("rpc");
        this.orm = useService("orm");
        this.actionService = useService("action");
    }

    async fetchCustomData() {
        // Option 1: Direct JSON-RPC call to a controller route
        const result = await this.rpc("/my_module/custom_route", {
            param1: "value",
        });

        // Option 2: ORM call (search_read, call, etc.) directly to a model
        const records = await this.orm.searchRead("res.partner", [['is_company', '=', true]], ["name", "email"]);
        
        // Option 3: Trigger a window action
        this.actionService.doAction({
            type: 'ir.actions.act_window',
            res_model: 'res.partner',
            view_mode: 'list,form',
            views: [[false, 'list'], [false, 'form']],
        });
    }
}
```