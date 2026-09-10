Dear {{firstName}},

We are pleased to inform you that your data submission request for the **{{study}}** study to the CRDC Submission Portal has been approved by the CRDC Submission Review Committee (SRC), with pending conditions. Please find the review comments and details on the pending conditions below.

{{reviewComments}}

**[Pending Conditions]:**

{{pendingConditions}}

{{#if omitSubmissionGuideInFooter}}
You will be notified once all pending conditions have been cleared. At that time, you may begin your data submission.

Thank you and we look forward to supporting your data submission.
{{else}}
You will be notified once all pending conditions have been cleared. At that time, you may begin your data submission. For guidance, please refer to the [Data Submission Instructions]({{submissionGuideURL}}).

If you have any questions, feel free to contact the CRDC Helpdesk at {{contactEmail}}.

Thank you for your submission, and we look forward to supporting your study.
{{/if}}

Sincerely,
CRDC Submission Portal
