# CRDC Submission Portal email content

HTML layouts and YAML copy constants for transactional emails sent by the CRDC Submission Portal.

## Layout

- [`email-templates/`](email-templates/) — Handlebars HTML layouts
- [`yaml/notification_email_values.yaml`](yaml/notification_email_values.yaml) — subjects, body fragments, sender, and committee constants

## How copy is structured

Most emails combine a YAML string with an HTML layout. The YAML holds subjects and paragraph text. The HTML holds greeting, optional sections (review comments, study fields, lists), and the signature.

In YAML, Data Hub substitutes `$placeholders` (for example `$study`, `$url`, `$helpDesk`). Some values include HTML such as `<b>` or `<a href="...">`. Keep that as written; do not convert these files to Markdown.

In HTML, Handlebars fills in values with `{{ name }}` (escaped) or `{{{ name }}}` (unescaped HTML). Conditionals use `{{#if }}` / `{{#each }}`. Helpers used in these templates include `markdownToHtml`, `isArray`, `and`, and `or`.

## HTML templates

| File | Role |
|------|------|
| [`notification-template.html`](email-templates/notification-template.html) | Generic notification: message parts, optional review comments, study name, extra paragraphs, user lists, and labeled additional info |
| [`notification-template-user.html`](email-templates/notification-template-user.html) | Account and request emails: access request, role change, and permissive-value inquiry (labeled `additionalInfo` or `pendingPV` blocks) |
| [`notification-template-submission.html`](email-templates/notification-template-submission.html) | New data submission created; notifies the Data Commons team |
| [`notification-template-edit-submission.html`](email-templates/notification-template-edit-submission.html) | Data submission configuration changed (submitter and/or model version) |
| [`notification-template-sr-reopen.html`](email-templates/notification-template-sr-reopen.html) | Submission Request reopened, including study/program and optional ownership change |
| [`notification-template-sr-inquire.html`](email-templates/notification-template-sr-inquire.html) | Submission Request additional information needed, with study fields and review comments |
| [`notification-template-SR-pending-conditions.html`](email-templates/notification-template-SR-pending-conditions.html) | Submission Request approved with one or more pending conditions |
| [`notification-template-pending-clear.html`](email-templates/notification-template-pending-clear.html) | Pending condition(s) cleared; submitter may begin data submission |
