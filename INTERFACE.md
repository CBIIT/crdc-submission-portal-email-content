# Data Hub email content implementation plan

This repository is the source of truth for transactional email copy. Data Hub loads YAML specifications and Markdown bodies from GitHub at runtime, caches each template by version, and renders them with send-time values. Changing copy does not require a Data Hub code deploy unless a **new `id`**, **new variable**, or **new named block** is introduced.

Use [`id`](#id) to select a template. Do not key sends on `notificationKey`; that field is a user-preference flag and is shared across multiple templates.

## Goals

1. Load every email spec and body from this GitHub repository without embedding copy in Data Hub.
2. Cache the latest successfully loaded version of **each** template so sends continue if GitHub is unreachable.
3. Render subjects and bodies with Handlebars, then send HTML plus a plain-text alternative.
4. Provide an admin-only test-email API that sends one template to the caller’s own address using placeholders, with **all** conditional copy visible.

## Non-goals

- Editing copy from Data Hub (authors change files in this repo).
- Using `notificationKey` as a template selector.
- Supporting Handlebars features the content files do not use (`{{#each}}`, `{{#with}}`, partials, `{{{triple-stash}}}`).

## Architecture

```text
GitHub repo (this content)
        │  fetch on startup + periodic refresh
        ▼
 Template loader ──► per-id cache (YAML + Markdown + source version)
        │
        ├── production send: look up id → real context → Handlebars → MIME
        └── test send: look up id → placeholders + all branches → caller inbox only
```

Data Hub keeps a compiled template object per `id`. The cache is the operational store after a successful load. GitHub is the publisher, not a dependency of every send.

---

## 1. Discover and parse templates

Scan these directories for `*.yaml` files:

- `submission-request/`
- `data-submission/`
- `account/`

Ignore `README.md`, `VARIABLES.md`, and this file. Each YAML file is one email. The Markdown body lives next to it, named by `contentFile`.

### YAML schema

| Field | Required | Type | Notes |
|--------|----------|------|--------|
| `id` | yes | string | Unique send key. Example: `data-submission.cfg-changed` |
| `contentFile` | yes | string | Path to the body, relative to the YAML file |
| `name` | yes | string | Human title (logging, admin UI) |
| `description` | yes | string | When the email is sent |
| `notificationKey` | yes | string | Preference key already used in Data Hub (`area:event`) |
| `subject` | yes | string | Subject line; may contain Handlebars `{{variables}}` |
| `variables` | yes | list of strings | Scalar keys Data Hub must supply (including unused ones) |
| `blocks` | no | list of strings | Named regions Data Hub injects as preformatted Markdown. Omit or use `[]` when none |

Reject a template (do not put it in the live cache) if:

- YAML is invalid
- `id` is missing or duplicated in the fetch
- `contentFile` does not resolve to a readable file
- `variables` or `blocks` contain names that are not strings

Do not infer the body path from the YAML basename. Always use `contentFile`.

### Example specification

```yaml
id: data-submission.cfg-changed
contentFile: cfg-changed.md
name: Data Submission Configuration Changed
description: Sent to the submitter when the assigned submitter and/or data model version on a submission changes.
notificationKey: data_submission:cfg_changed
subject: "Your CRDC Data Submission Configuration Has Changed"
variables:
  - firstName
  - portalURL
  - studyName
  - prevModelVersion
  - newModelVersion
  - prevSubmitterName
  - newSubmitterName
```

### Load the Markdown body

1. Take the directory of the YAML file.
2. Join `contentFile` (reject `..` and absolute paths).
3. Read the file as UTF-8 text. That string is the Handlebars template for the body.
4. Keep YAML and Markdown together as one compiled template object, keyed by `id`.

---

## 2. Fetch from GitHub and cache by version

Load templates from the configured GitHub repository (owner, name, ref). Treat a successful fetch as a new **source version** (git commit SHA of the ref, or equivalent ETag / tree SHA). Persist that version with each cached template.

### Cache contents (per `id`)

| Field | Purpose |
|--------|---------|
| `id`, `name`, `description`, `notificationKey`, `subject` | Spec metadata |
| `variables`, `blocks` | Context contract |
| `bodyTemplate` | Markdown Handlebars source |
| `sourceVersion` | Commit SHA (or equivalent) of the fetch that produced this copy |
| `cachedAt` | When this version was stored |

Use durable cache storage (not only process memory) so a restart can serve the last good copy if GitHub is still down. Memory may hold the compiled Handlebars functions derived from the durable cache.

### Refresh behavior

1. **Startup:** fetch GitHub. On success, validate, compile, and replace the cache for every `id` in that version. Record `sourceVersion`.
2. **Periodic refresh:** same as startup (interval via config). Do not block sends on refresh.
3. **GitHub unreachable, timeout, rate-limited, or invalid HTTP:** keep the existing cache. **Log a warning** that includes:
   - that GitHub could not be reached (or the fetch failed)
   - the cached `sourceVersion` still in use
   - which templates remain served from cache (all current `id`s, or a count plus version)
4. **Partial / invalid repo:** do not wipe a good cache for templates that fail to parse in a bad fetch. Log errors for invalid files. Only replace a cached `id` when the new version of that template parses successfully.
5. **No GitHub and empty cache:** fail startup (or fail the first send) with an error. There is no fallback copy.
6. **Removed templates:** if a successful fetch no longer contains an `id`, drop it from the live map after the new version is committed. Do not drop it because a *failed* fetch returned nothing.

Sends always read the cache. They never call GitHub on the hot path.

### Logging

| Event | Level |
|--------|--------|
| Successful fetch and cache update | info (`sourceVersion`, template count) |
| GitHub unreachable / fetch failed; serving cache | **warn** (`sourceVersion` in use, failure reason) |
| Parse failure for a file | error (path, reason); other templates unchanged |
| Send used cached copy while last refresh failed | warn once per refresh-failure window, not once per email |

---

## 3. Render at send time (production)

Pipeline:

1. Look up the template by `id` in the cache. If missing, fail the send.
2. Build a context object with every name in `variables` and `blocks`.
3. Render `subject` with Handlebars (plain text; not Markdown).
4. Render the Markdown body with Handlebars (still Markdown). **Evaluate** `{{#if}}` as usual.
5. Convert the rendered body from Markdown to HTML for the email MIME part.
6. Send HTML plus a plain-text alternative (strip Markdown or reuse the rendered Markdown).
7. Honor user notification preferences using `notificationKey` **after** template selection, not as the selector.

### Context

- Keys listed in `variables` are scalars (string, number, boolean, or empty).
- Keys listed in `blocks` are Markdown fragments produced by Data Hub (lists, labeled rows, comment text). Insert them as already-formatted Markdown, not as escaped HTML.
- Missing optional values must be empty string, `null`, or omitted so `{{#if}}` is false. Do not pass the literal `"undefined"`.
- Historical names are not always camelCase. Use the names in that template’s YAML, not a renamed internal model. See [`VARIABLES.md`](VARIABLES.md).

### Handlebars dialect

Bodies and subjects use [Handlebars](https://handlebarsjs.com/), not Mustache. Mustache cannot evaluate `and` / `or` subexpressions.

Required support:

| Syntax | Meaning | Used in |
|--------|---------|---------|
| `{{name}}` | Substitute a variable or named block | all templates |
| `{{#if name}}` … `{{/if}}` | Include the block when `name` is truthy | `reopened.md` |
| `{{#if name}}` … `{{else}}` … `{{/if}}` | If/else | `approved-pending-multiple.md` |
| Nested `{{#if}}` | Inner conditionals | `cfg-changed.md` |
| `{{#if (and a b)}}` | True when both `a` and `b` are truthy | `cfg-changed.md` |
| `{{#if (or a b)}}` | True when either is truthy | `cfg-changed.md` |
| Nested subexpressions | e.g. `(or (and a b) (and c d))` | `cfg-changed.md` |

`and` and `or` are **not** Handlebars builtins. Register helpers equivalent to:

- `and`: every argument is truthy
- `or`: at least one argument is truthy

Truthy follows Handlebars: `false`, `null`, `undefined`, `0`, `""`, and empty arrays are falsy. For configuration-change emails, pass a previous/new pair only when that field actually changed; leave the unused pair unset so those sections are omitted.

Do not HTML-escape during Handlebars render. Output is Markdown. Escaping belongs in Markdown→HTML (plain variables escaped; named blocks treated as Markdown).

Templates do not use `{{#each}}`, `{{#with}}`, partials, or `{{{triple-stash}}}`. Do not add those in content files without a Data Hub change.

### Markdown subset

After Handlebars, the body is Markdown. Support at least:

- Paragraphs
- Bullet and numbered lists
- `**bold**` and `*italic*`
- `[label](url)` links, including URLs that came from `{{variables}}`

Do not expect raw HTML in the content files. Named blocks may contain Markdown that Data Hub generated (for example a bullet list for `pendingConditions`).

### Named blocks vs variables

| | `variables` | `blocks` |
|--|-------------|----------|
| Source | Scalar send-time fields | Structured snippets Data Hub builds |
| Template token | `{{firstName}}` | `{{reviewComments}}` |
| Typical content | names, URLs, versions | comments, extra rows, user lists |
| If unused | still listed on the spec | omit the `blocks` key |

A spec may list a block that the Markdown does not reference. Still accept it; do not fail the send.

Known block names: `reviewComments`, `pendingConditions`, `additionalInfo`, `additionalMsg`, `users`, `pendingPV`. Meanings are in [`VARIABLES.md`](VARIABLES.md).

### Worked example: `data-submission.cfg-changed`

This template is one send covering two optional change types. Data Hub chooses **one** `id` (`data-submission.cfg-changed`) and fills only the pairs that apply.

| Situation | Set | Leave unset / empty |
|-----------|-----|---------------------|
| Submitter reassigned | `prevSubmitterName`, `newSubmitterName` | `prevModelVersion`, `newModelVersion` |
| Data model version changed | `prevModelVersion`, `newModelVersion` | `prevSubmitterName`, `newSubmitterName` |
| Both | all four | — |

Always set `firstName`, `portalURL`, and `studyName`.

The body then:

1. Shows “Submitter: {{newSubmitterName}}” only when both submitter names are present.
2. Shows a **[Changes]** section if either pair is present.
3. Shows responsibility copy only for a submitter change.
4. Shows validation-reset copy only for a model-version change.

Pseudo-code:

```text
spec = yaml.load("data-submission/cfg-changed.yaml")
bodyTemplate = read(dir(specFile) + spec.contentFile)
ctx = {
  firstName, portalURL, studyName,
  prevModelVersion, newModelVersion,      # empty if unchanged
  prevSubmitterName, newSubmitterName     # empty if unchanged
}
subject = handlebars(spec.subject, ctx)
markdown = handlebars(bodyTemplate, ctx)
html = markdownToHtml(markdown)
```

---

## 4. Admin test-email API

Admins need to preview a live template in a real inbox without notifying other users.

### Behavior

- Caller authenticates with their existing Data Hub token.
- Request names a template `id`.
- Data Hub renders that template with **placeholder values** (not production data).
- The message is sent **only** to the authenticated user’s email address. Do not use the template’s usual recipient list, CC, or notification-preference routing.
- Prefix the subject with a test marker, for example `[TEST] `, so it is obvious in the inbox.
- Skip `notificationKey` preference checks for this send.

### Authorization

Limit the operation to **admin** users **and** require a dedicated permission (for example `email:test-send`) so the capability can be granted or revoked without making every admin able to trigger mail.

| Check | On failure |
|--------|------------|
| Valid token | 401 / unauthenticated |
| User has admin role | 403 |
| User has `email:test-send` (or the chosen permission name) | 403 |
| Template `id` exists in cache | 404 |

Do not allow a target address in the request. The only recipient is the token’s user. Rate-limit per user to reduce abuse (config; suggested starting point: a small number of sends per minute).

Log each test send at info: user ID, email, template `id`, `sourceVersion`. Do not log full rendered bodies if they might contain future PII; placeholders should be non-sensitive.

### API shape

Expose a GraphQL mutation (REST equivalent is acceptable if that is the local pattern). Example:

```graphql
mutation testEmail($id: String!) {
  testEmail(id: $id) {
    success
    templateID
    sourceVersion
    recipientEmail
  }
}
```

```json
{
  "id": "data-submission.cfg-changed"
}
```

`recipientEmail` is the caller’s address, returned so the UI can confirm where the message went. Do not accept a recipient argument.

### Placeholder context

Build the Handlebars context from the spec’s `variables` and `blocks` only. Do not load studies, submissions, or user records.

| Kind | Placeholder |
|------|-------------|
| Person / name | `[First Name]`, `[Submitter Name]`, etc. derived from the key |
| Email | `placeholder@example.com` |
| URL | a stable docs or portal example URL from config |
| IDs | `[Submission ID]`, `[CDE ID]` |
| Counts / days | a small obvious number such as `7` |
| Booleans used as flags | treat as true for production-like substitution; **do not** rely on this alone to reveal copy (see [all conditional content](#all-conditional-content)) |
| Named blocks | short sample Markdown (bullet list, labeled rows, or a comment paragraph) labeled as sample data |

Unknown keys not in [`VARIABLES.md`](VARIABLES.md) still get a bracketed placeholder from the key name so new variables remain visible without a Data Hub change to the placeholder table.

### All conditional content

Production sends evaluate `{{#if}}` and omit false branches. **Test sends must include every branch** so reviewers see copy that would only appear for some events.

Implement a test-only render path:

1. Walk the compiled Handlebars AST (or an equivalent preprocessor).
2. For `{{#if}}` … `{{/if}}` with no `{{else}}`, include the inner content (do not omit it).
3. For `{{#if}}` … `{{else}}` … `{{/if}}`, include **both** the consequent and the `{{else}}` content, in that order.
4. Nested conditionals are included the same way (for example every section of `cfg-changed.md`).
5. Still substitute placeholders for `{{name}}` tokens inside those branches.

Do not invent a second production send. This path is only for the test API.

Consequence for `approved-pending-multiple.md`: the test body includes the imaging-style footer **and** the default footer (with submission guide link and helpdesk). That duplication is intended.

Consequence for `cfg-changed.md`: the test body includes submitter rows, model-version rows, and both follow-up paragraphs.

Consequence for `reopened.md`: the test body includes the ownership-change section.

---

## Contract with Data Hub

- **`id` is the API.** Wiring a send to `data-submission.cfg-changed` is a code change. Renaming `id` is a breaking change.
- **`notificationKey` is not unique.** Example: `submission_request:reviewed` is shared by approved, rejected, inquire, and pending-approval variants. `data_submission:expiring` is shared by first and final reminders.
- **Copy is not an API.** Wording in `.md` and `subject` may change without a code change.
- **New `{{name}}` tokens require a code change** so Data Hub can populate the production context. Add the name to that file’s `variables` or `blocks` in the same PR. Test sends can still show a generic placeholder for unknown keys.
- **Several templates may share one preference key** but differ in `id` and body. Branch in application code on event/outcome, then load the matching `id`.
- Values may include trailing punctuation or wrapping (study name with a comma, email with a period). Do not add extra punctuation in code if the template already includes it; see [`VARIABLES.md`](VARIABLES.md).
- **GitHub outage must not drop mail** if a cache exists. Warn in logs and keep sending the last good version.
- **Test mail never leaves the caller’s inbox** and never uses live entity data.

---

## Implementation sequence

1. **Loader and validator** — scan dirs, parse YAML, resolve `contentFile`, compile Handlebars with `and` / `or`.
2. **Durable per-id cache** — store spec, body, `sourceVersion`; memory compile on top.
3. **GitHub fetch + refresh job** — startup and interval; warn and keep cache on failure.
4. **Production render/send** — existing notification flows keyed by `id`.
5. **Permission** — add `email:test-send` (name TBD in Data Hub) and attach it only to the admin role by default.
6. **Test mutation** — authz, placeholders, all-branches render, send to caller only, rate limit, audit log.
7. **Config** — repo owner/name/ref, fetch timeout, refresh interval, cache location, test rate limit, example portal URL for placeholders.

## Acceptance

- Changing Markdown or `subject` in GitHub appears in the next successful refresh without a Data Hub deploy.
- With GitHub blocked, a send still uses the last cached version and a warning is logged.
- Cold start with no cache and no GitHub fails clearly.
- Admin with the test permission can send `id` to themselves only; other users cannot.
- Test body for `data-submission.cfg-changed` includes both submitter and model-version copy.
- Test body for `submission-request.approved-pending-multiple` includes both footer variants.
- Production `cfg-changed` still omits unchanged pairs.
