# CRDC Data Hub email content

Proposed templates for transactional emails. These files are **not wired into sending** yet. After review they are intended to move to an external GitHub repository that Data Hub can load without a code deploy.

## Layout

- `submission-request/` — Submission Request Form (SRF) emails
- `data-submission/` — data submission lifecycle emails
- `account/` — access request, role change, and account status emails
- [`VARIABLES.md`](VARIABLES.md) — catalog of placeholder names and what they mean

Each email is a **YAML specification** plus a **Markdown body**. The YAML `id`, `name`, filename, and folder identify which message you are editing.

## File format

Specification ([`submission-request/approved.yaml`](submission-request/approved.yaml)):

```yaml
id: submission-request.approved
contentFile: approved.md
name: Submission Request Approved
description: When this email is sent.
notificationKey: submission_request:reviewed
subject: "CRDC Submission Request Decision"
variables:
  - firstName
  - study
  - contactEmail
blocks:
  - reviewComments
```

Body ([`submission-request/approved.md`](submission-request/approved.md)), previewed as Markdown:

```markdown
Dear {{firstName}},

Body copy with **bold**, lists, and [links](https://example.com).

{{reviewComments}}

Sincerely,
CRDC Submission Portal
```

`contentFile` is a path relative to the YAML file (same directory, same basename).

### Specification fields

| Field | Purpose |
|--------|---------|
| `id` | Stable key. Do not rename without a Data Hub code change. |
| `contentFile` | Markdown file with the email body |
| `name` | Human title |
| `description` | When the email is sent |
| `notificationKey` | User notification preference key in Data Hub |
| `subject` | Subject line; may include `{{variables}}` |
| `variables` | Every scalar Data Hub provides for this send, including unused ones |
| `blocks` | Named system-injected regions (omit if none) |

### Body

Use Markdown paragraphs, bullet/numbered lists, **bold**, *italic*, and `[text](url)`.

Use `{{camelCaseName}}` (or the historical name listed in the spec) for Data Hub values. Do not use HTML tags.

Named blocks (`{{reviewComments}}`, `{{additionalInfo}}`, and others) render structured content. Optional copy can use `{{#if flag}}` … `{{/if}}`.

## Variable names

Lists in each spec match **keys Data Hub currently passes** at send time (including unused ones). Some historical names are not camelCase (`study` vs `studyName`, `title` vs `submissionName`, `dataCommonName` vs `dataCommonsName`). See [`VARIABLES.md`](VARIABLES.md).

Some values currently include trailing punctuation or wrapping (for example a comma after a study name, or a period after an email). That comes from Data Hub, not from these files.
