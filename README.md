# NextForm

## What is NextForm?

NextForm is an application that allows you to easily create dynamic, ephemeral, UI-based forms in ServiceNow.

Previously, to create forms displayed in the UI of ServiceNow you would either need to create a table, or create a record producer. Sometimes these methods are perfectly fine; other times they can be more complex than needed. For example:

- If you create a table, you get the overhead that comes with it, such as ACLs, views, related lists, UI actions, and so on.
- If you create a record producer, you can only use it to create new records. Record producers are not designed to open existing records and edit those values.

NextForm lets you define a form that exists for the UI flow: values can be pre-populated from the instance (via your generator script and optional input), and on submit they can be handled by your processor script. How you generate the form and what you do with the result is up to the developer using NextForm.

![NextForm Overview](images/nextform-overview.png)

NextForm’s UI Builder integration is driven by its **controller** macroponent. Using its [preset](https://docs.servicenow.com/bundle/tokyo-application-development/page/administer/ui-builder/concept/presets.html), you can attach it to a Form component: add the Form component to your UI Builder page and connect it to a NextForm when prompted.

![NextForm "Preset" prompt](images/form-component-preset.png)

In the **NextForm Sys ID** field, enter the **Sys ID** of a record in the **NextForms** [`x_snc_nf_nextform`] table and click **Use**. Each NextForm record references a **Generator** script and a **Processor** script (Business Rules or similar scriptable records evaluated by `GlideScopedEvaluator`).

## Runtime flow (high level)

1. **Load:** The controller calls the scripted REST API to run the generator for the configured NextForm sys_id. The response JSON contains `screens` (layout) and `fields` (a map keyed by field name). Client state keeps the full layout in `nextFormData` but omits the `fields` map from that object to avoid duplication; the Form component consumes `fields` and `currentSections` separately.
2. **Navigate:** Screen navigation updates `currentSections`, `currentScreenNumber`, and flags such as `canProcess` (true on the last screen when not yet processed).
3. **Process:** Emitting **\[NextForm] Process** builds a payload from client field state, POSTs to the process API, and runs your processor script with `nfData` (and optional `metadata`).

## Generators

Generators build the JSON that the UI renders. They are evaluated server-side and **must return an object that implements `toJSON()`** returning the shape the client expects. The built-in **`x_snc_nf.NextForm`** API does exactly that; you can also return any compatible object.

When the form is loaded from **UI Builder**, optional **Form data** (`formData` on the controller) is sent as the generator’s `input` variable (POST body). The same `input` is available when you call **`POST /api/x_snc_nf/nextform/generate/{nextform_id}`** with a JSON body (see [Scripted REST](#scripted-rest) below). **`GET /api/x_snc_nf/nextform/generate/{nextform_id}`** runs the generator with no `input`.

### The NextForm generator API (`x_snc_nf.NextForm`)

#### `new NextForm()`

Creates a `NextForm`. No parameters.

#### `NextForm.addScreen()`

Adds a screen (wizard step). No parameters.

#### `NextForm.addRow()`

Adds a row to the current screen for vertical layout. If there is no screen yet, a screen is created first. No parameters.

#### `NextForm.addColumn()`

Adds a column to the current row for horizontal layout. If there is no row yet, a row is created first. No parameters.

#### `NextForm.addField(x_snc_nf.NextField field)`

Adds a field to the current column and registers it in the form’s `fields` collection. The field **`name`** must be unique across the entire form.

#### `x_snc_nf.NextField` — `new NextField(type, label, name, options)`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `type` | Yes | Logical type string (see [Field types](#field-types)). |
| `label` | Yes | Label shown in the UI. |
| `name` | Yes | Stable identifier; must be unique across the form. |
| `options` | No | Type-specific options (e.g. `maxLength`, `choices`, `value`, `displayValue`). |

### Field types

`NextField` maps the `type` argument to the underlying Form field definition. Supported **`type`** values in code include:

`string`, `integer`, `choice`, `dateTime`, `boolean`, `currency`, `decimal`, `date`, `list`, `time`, `html`, `reference`, `url`, `label`, `annotation`.

Any other `type` falls back to **`string`** behavior (same as an unknown type).

Common options:

- **String-like:** `maxLength` (default 40 for string if omitted), etc.
- **Choice:** `choices` array of `{ displayValue, value }`, plus optional `value` / `displayValue` for defaults.

### Example generator script

```javascript
(function generateNextFormObject() {

    return new x_snc_nf.NextForm()
        .addScreen()
        .addRow().addColumn()
        .addField(new x_snc_nf.NextField('string', 'What is your given name?', 'name_given'))
        .addRow().addColumn()
        .addField(new x_snc_nf.NextField('string', 'What is your family name?', 'name_family'))
        .addField(new x_snc_nf.NextField('choice', 'What is your age band?', 'age', {
            choices: [{
                displayValue: 'Under 18',
                value: 'under_18'
            }, {
                displayValue: '18 and Over',
                value: 'over_18'
            }],
            value: 'under_18',
            displayValue: 'Under 18'
        }))
        .addColumn()
        .addField(new x_snc_nf.NextField('dateTime', 'When did you last sleep?', 'sleep'))
        .addField(new x_snc_nf.NextField('dateTime', 'When did you last wake up?', 'wake'))
        .addScreen()
        .addRow().addColumn()
        .addField(new x_snc_nf.NextField('string', 'Tell me about yourself', 'details', {
            maxLength: 4000
        }))
        .addScreen()
        .addRow().addColumn()
        .addField(new x_snc_nf.NextField('boolean', 'Would you like to receive marketing communications?', 'marketing_allowed'));

})();
```

## Processors

Processors run when the user submits from the last screen. The evaluator provides:

- **`nfData`** — For each submitted field, an entry with (at least) `name`, `value`, and `displayValue` (see client build in **Submit for Processing**).
- **`metadata`** — Optional object: set **Processor Metadata** on the controller to pass stringified key/value pairs through to the processor.

If **Only process edited values** (`processEditedOnly`) is enabled on the controller, only fields that differ from their `originalValue` are included in `nfData` (non–data fields such as labels are excluded by the client).

### Example processor script

`nfData` and, when the client sends it, `metadata` are available as evaluator variables (see `NFAPIHelper.process`).

```javascript
(function processNextFormResult() {

    gs.info('NextForm processed: ' + JSON.stringify(nfData));

    if (nfData.name_family) {
        gs.info("The user's family name was: " + nfData.name_family.displayValue);
    }

    if (typeof metadata !== 'undefined') {
        gs.info('Metadata: ' + JSON.stringify(metadata));
    }

})();
```

## Scripted REST

Base path: **`/api/x_snc_nf/nextform`**

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/generate/{nextform_id}` | Run the generator with no `input`. |
| `POST` | `/generate/{nextform_id}` | Run the generator; optional `input` on `request.body.data` (same shape the scripted REST handler expects). |
| `POST` | `/process` | Run the processor with `nextform_id`, `nextform_data`, and optional `metadata` on `request.body.data`. |

Errors from `NFAPIHelper` use HTTP problem semantics (e.g. missing NextForm, missing generator/processor, or script failures).

## NextForm controller (UI Builder)

The controller is added when you apply the NextForm preset to the Form component. Configure it in **Data Resources** / component properties.

### Controller properties (inputs)

| Property | Description |
|----------|-------------|
| **NextForm SysID** | Sys ID of the `x_snc_nf_nextform` record (required). |
| **Form data** | JSON passed to the generator as `input` when the layout is fetched. |
| **Processor Metadata** | JSON available to the processor as `metadata`. |
| **Only process edited values** | When true, the client sends only changed data fields in `nfData` on process. |

### Output properties

| Property | Type | Description |
|----------|------|-------------|
| `nextFormData` | JSON | Layout from the generator (e.g. `screens`). The `fields` map is **not** duplicated here; use the `fields` output. |
| `fields` | JSON | Field map for the Form component’s `fields` input. |
| `currentSections` | JSON | Sections for the current screen, for the Form component’s `sections` input. |
| `currentScreenNumber` | Integer | Current screen, **1-based** (for display). |
| `currentScreenIndex` | Integer | Current screen, **0-based** (for logic). |
| `screenCount` | Integer | Total number of screens. |
| `canGoFirst` | Boolean | Whether the user can go to the first screen. |
| `canGoLast` | Boolean | Whether the user can go to the last screen. |
| `canGoNext` | Boolean | Whether the user can go to the next screen. |
| `canGoPrevious` | Boolean | Whether the user can go to the previous screen. |
| `canGoIndex` | Boolean | Whether index-based navigation is meaningful (more than one screen). |
| `isLoading` | Boolean | Loading in progress (initial load, processing, etc.). |
| `canReset` | Boolean | Whether reset/reload is allowed (depends on screen and processed state). |
| `canProcess` | Boolean | Whether submit is allowed (last screen and not yet processed). |
| `isLoaded` | Boolean | Initial load finished. |
| `isProcessed` | Boolean | Form has been successfully processed. |

### Handled events (UX Event names)

| Label | Description | Parameters |
|-------|-------------|------------|
| **\[Screen] Change current screen** | Jump to a screen by **0-based** index. | **Screen index** (`screenIndex`): integer, first screen is `0`. |
| **\[NextForm] Process** | Submit field values to the processor script. | — |
| **\[Field] Set value** | Set a field’s `value` and `displayValue`. | Field name, value, display value. |
| **\[NextForm] Reload** | Reload generator output and reset client state. | — |
| **\[Screen] Go to first screen** | Go to the first screen. | — |
| **\[Screen] Go to last screen** | Go to the last screen. | — |
| **\[Screen] Go to next screen** | Next screen if allowed. | — |
| **\[Screen] Go to previous screen** | Previous screen if allowed. | — |

### Dispatched events

| Label | When |
|-------|------|
| **\[NextForm] Processing completed** | Server processing finished successfully. |
| **\[NextForm] Field value changed** | A field value changed in the UI. |
| **\[NextForm] Processing started** | Submit has been triggered. |
| **\[NextForm] Loading completed** | Initial layout and field state are ready. |
