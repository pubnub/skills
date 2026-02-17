# Anonymous Employee Satisfaction Survey

## Problem/Feature Description

A large technology company wants to run an anonymous employee satisfaction poll during their all-hands meeting. The poll will ask employees to rate the company's direction on a set of topics. Because the topics are sensitive (e.g., satisfaction with leadership, compensation fairness), employees must not be identifiable from their votes. At the same time, the system must prevent any employee from voting multiple times, and it needs to detect and block any attempt to stuff the ballot through rapid-fire automated submissions.

The engineering team needs to build the server-side vote processing logic for this anonymous poll using PubNub Functions and KV Store. The solution must ensure that individual votes cannot be traced back to the employee who cast them, while still reliably preventing duplicates. It must also include protections against automated vote-stuffing -- if someone scripts rapid submissions, the system should detect and block them.

Build the PubNub Function that processes these anonymous votes with fraud protection.

## Output Specification

Produce the following files:

1. `anonymous-vote-function.js` - A PubNub Before Publish Function that handles anonymous voting: it must ensure voter privacy by not storing identifiable voter IDs, while still preventing duplicate votes. Include fraud detection logic that catches rapid-fire submissions.

2. `fraud-detection.js` - A module containing the fraud detection helpers used by the main function, including rate limiting and any session-based detection mechanisms.

3. `security-design.md` - A brief document explaining how the anonymization works, why voters cannot be identified from the stored data, and how duplicate prevention and fraud detection operate together.
