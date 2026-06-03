# Email Templating

An event model for defining reusable email templates. A template carries an id,
a human description, a default subject and body, and declared placeholder
variables (required and optional). Templates are seeded as part of system
initialization and then surfaced through a list view and a per-template detail
view.

## Actors and lanes

The board uses the standard event-modeling lanes: a single **Actor** lane, an
**Interaction** lane (screens, command, read models), a **Swimlane** for
events, and an empty **Spec Lane** (no specifications have been authored yet).

## Slices (left to right)

### 1. Define Email Template (Initialization Task)

The **System Initialization Task Processor** automation issues
`Define Email Template`, emitting `Email Template Defined`. Rather than being
driven by a user screen, template creation is wired off the system bootstrap —
this is the "Initialization Task" that ships a baseline set of templates with
the system. (The same processor backs the separate `System_Initialization`
model.)

A template carries:

- `email_template_id` (e.g. `welcome`) — the stable identifier
- `description` (e.g. `Welcome email`)
- `default_subject` (e.g. `Welcome`)
- `default_body` (e.g. `Hi {{name}}, your link: {{link}}`)
- `required_placeholders` — list of variables a render *must* supply
  (e.g. `["link"]`)
- `optional_placeholders` — list of variables a render *may* supply
  (e.g. `["name"]`)

The body and subject are `{{mustache}}`-style templates; the required/optional
placeholder lists declare which variables the body expects.

### 2. Email Templates (list read model)

`Email Template Defined` populates the **Email Templates** list read model
(`email_template_id`, `description`). It backs the **List Email Templates**
screen — a catalogue of every defined template.

### 3. Email Template (detail read model)

The same event also populates the **Email Template** read model
(`email_template_id`, `subject`, `body`), which maps `default_subject` →
`subject` and `default_body` → `body`. It backs the **View Email Template**
screen, the per-template detail view.

## Key information

- **Context**: `INTERNAL` across all elements.
- **Created at initialization, not by a user**: there is no "create template"
  screen. The **System Initialization Task Processor** automation issues
  `Define Email Template`, so the template set is provisioned as part of system
  bootstrap.
- **One event, two read models**: `Email Template Defined` fans out to a list
  read model (the catalogue) and a single-template read model (the detail
  view), each backing its own screen.
- **Placeholders are declared, not just embedded**: alongside the
  `{{name}}` / `{{link}}` tokens in the body, the template records explicit
  `required_placeholders` and `optional_placeholders` lists, making the
  template's variable contract first-class data.
- **Status**: the **Spec Lane** is empty — no Given/When/Then specifications
  have been written (e.g. validating that a body only references declared
  placeholders, or that required placeholders are present).
