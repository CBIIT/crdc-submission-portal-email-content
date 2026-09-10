Dear {{firstName}},

Your data submission has been updated by the CRDC team in the CRDC Submission Portal. Please log in to the [Data Submission Portal]({{portalURL}}) site to review the changes and take any necessary actions.

**[Data Submission]:**

{{#if (and prevSubmitterName newSubmitterName)}}
- Submitter: {{newSubmitterName}}
{{/if}}
- Study: {{studyName}}

{{#if (or (and prevSubmitterName newSubmitterName) (and prevModelVersion newModelVersion))}}
**[Changes]**

{{#if (and prevSubmitterName newSubmitterName)}}
- Submitter: {{prevSubmitterName}} → {{newSubmitterName}}
{{/if}}
{{#if (and prevModelVersion newModelVersion)}}
- Data Model Version: {{prevModelVersion}} → {{newModelVersion}}
{{/if}}
{{/if}}

{{#if (and prevSubmitterName newSubmitterName)}}
As the newly assigned Submitter, you are now responsible for completing and managing this submission in the portal.
{{/if}}

{{#if (and prevModelVersion newModelVersion)}}
As the data model version has been changed, all previous validation results have been reset. You are required to rerun validation against the new model version.
{{/if}}

If you have any questions or need assistance, please contact your assigned Data Concierge.

Sincerely,
CRDC Submission Portal
