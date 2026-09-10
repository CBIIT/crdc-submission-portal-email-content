# Email template variables

Names below are the keys Data Hub passes today. Each template file lists only the keys available for that send, including unused ones.

## Greeting and chrome

| Name | Meaning |
|------|---------|
| `firstName` | Greeting name (person, committee, “Admins”, data commons team, etc.) |
| `senderName` | Sign-off name when it is not the default portal string |

## Study, program, PI

| Name | Meaning |
|------|---------|
| `study` | Study label used in several SRF YAML strings (often the study name) |
| `studyName` | Study name (sometimes passed with trailing punctuation) |
| `studyAbbreviation` | Study abbreviation (`NA` when missing) |
| `studyFullName` | Combined study display for DCP “new submission” email |
| `programName` | Program name |
| `programAbbreviation` | Program abbreviation |
| `pi` | Principal investigator (wording may include program context) |

## URLs and contacts

| Name | Meaning |
|------|---------|
| `url` | Submission Portal URL |
| `portalURL` | Same portal URL under a different key (config-changed and pending-cleared) |
| `submissionGuideURL` | Data Submission Instructions URL |
| `helpDesk` | Help desk / official contact for some user emails |
| `contactEmail` | Help desk, concierge, or conditional-approval contact |
| `contactName` | Concierge or contact display name |
| `officialEmail` | Official CRDC email |
| `techSupportEmail` | Tech support address |
| `conciergeName` | Data Concierge name |
| `conciergeEmail` | Data Concierge email |
| `primaryContactName` | Data Concierge on a new submission (or “not assigned”) |

## SRF / application

| Name | Meaning |
|------|---------|
| `inactiveDays` | Inactivity threshold in days |
| `remainDays` | Days remaining before SRF deletion |
| `canceledNameBy` | Who canceled the SRF |
| `isOwnershipChanged` | True when reopen assigned a new owner |
| `isMultiplePendingConditions` | True when several pending conditions apply |
| `omitSubmissionGuideInFooter` | Pending-approval footer variant (imaging) |
| `omitDataSubmissionInstructionsOnly` | Pending-approval footer variant (data model) |

## Data submission

| Name | Meaning |
|------|---------|
| `submissionName` | Data submission name (sometimes with trailing punctuation) |
| `submissionID` | Data submission ID |
| `title` | Submission name in expiration subjects/bodies |
| `dataCommonsName` | Data commons display name |
| `dataCommonName` | Same idea; used in some subjects and DCP emails |
| `canceledBy` | User who canceled the submission |
| `withdrawnByName` | User who withdrew the submission |
| `withdrawnByEmail` | Email of the user who withdrew |
| `expiredDays` | Days until expiration |
| `pastDays` | Days since last access |
| `days` | Days unused in the final submission reminder |
| `prevModelVersion` | Previous data model version (config change) |
| `newModelVersion` | New data model version |
| `prevSubmitterName` | Previous submitter display name |
| `newSubmitterName` | New submitter display name |

## Account and PV request

| Name | Meaning |
|------|---------|
| `userName` | Requesting user’s display name |
| `accountType` | Identity provider / account type |
| `email` | Account email |
| `role` | Role or requested role |
| `dataCommons` | Assigned data commons |
| `studies` | Assigned study names |
| `institution` | Institution on role-change emails |
| `institutionName` | Institution on access-request emails |
| `submitterName` | PV request submitter |
| `submitterEmail` | PV request submitter email |
| `nodeName` | Model node |
| `property` | Model property |
| `CDEId` | CDE identifier |
| `value` | Requested permissive value |
| `comment` | PV justification |

## Named blocks

| Name | Meaning |
|------|---------|
| `reviewComments` | Reviewer comments (Markdown from Data Hub) |
| `pendingConditions` | Bullet list of pending-approval conditions |
| `additionalInfo` | Label/value rows (access request, role change, release) |
| `additionalMsg` | Extra paragraphs (final submission expiration) |
| `users` | Deactivated user list (admin batch email) |
| `pendingPV` | Permissive-value request detail rows |
