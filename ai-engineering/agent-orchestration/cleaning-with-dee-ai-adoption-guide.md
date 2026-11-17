# Cleaning with Dee: AI Adoption & CRM Architecture

This guide outlines a practical path to grow Cleaning with Dee from a cleaning-service website into an AI-assisted customer and partnership management platform (CRM + AI agents). It centers on an administrative dashboard and human-controlled customer and stakeholder communication without treating AI output as an autonomous business decision.

> **Scope and assumptions:** This is a proposed architecture, not a description of a deployed system or legal advice. The Firestore collections and fields below are illustrative starting points; adapt them to the organization's actual processes, permissions, and retention needs. For general AI adoption, model selection, RAG, prompting, and evaluation concepts, see the [AI Solution Adoption & LLM Engineering Guide](../llm/ai-solution-adoption-and-llm-engineering-guide.md).

## Contents

- [1. Product goals and suggested stack](#1-product-goals-and-suggested-stack)
- [2. System architecture](#2-system-architecture)
- [3. CRM agents and boundaries](#3-crm-agents-and-boundaries)
- [4. Illustrative Firestore collections and fields](#4-illustrative-firestore-collections-and-fields)
- [5. Human-reviewed email workflow](#5-human-reviewed-email-workflow)
- [6. UK privacy and electronic-marketing safeguards](#6-uk-privacy-and-electronic-marketing-safeguards)
- [7. Illustrative Flutter call](#7-illustrative-flutter-call)
- [8. Phased MVP priorities](#8-phased-mvp-priorities)
- [Official references](#official-references)

## 1. Product goals and suggested stack

The proposed product is a Flutter Admin Dashboard with areas for customer, partnership, community, recruiter, and executive relationship management. It helps authorized staff review records, prepare useful summaries and drafts, and coordinate work across Cleaning with Dee's business relationships. The dashboard should make it clear which information is verified, which content was AI-generated, and what still needs human review.

| Layer | Suggested starting point | Responsibility |
| --- | --- | --- |
| Admin interface | Flutter | Authenticated staff workflows, record review, draft editing, approval, and status visibility |
| Identity | Firebase Authentication | Sign-in and identity token issuance; application roles still require server-side authorization |
| CRM data | Cloud Firestore | Operational records, references between records, workflow state, and narrowly scoped audit metadata |
| Trusted application backend | Firebase Cloud Functions or another authenticated server backend | Validate identity and roles, enforce business rules, read/write privileged records, call model services, and control email sending |
| Model integration | Gemini through a currently supported server-side API; Firebase AI Logic may be considered for supported app-mediated use cases | Optional summarization, classification, and draft generation, subject to current model, region, account, privacy, and service limits |
| Email provider | An organization-approved SMTP or other transactional/marketing email service | Send only approved messages and return delivery/status information to the backend |
| Supporting materials | Google Drive for working documents; Notion for operational procedures; Sheets for limited exports/analysis | Keep Firestore as the CRM source of truth; do not copy the same contact records into multiple tools |

This is a suggested stack, not a requirement. Firebase client SDK/API availability, quotas, model access, pricing, free-tier terms, and provider data handling can change. Check current product documentation, regional availability, service terms, and data-processing terms before implementation; do not assume any component or usage level will remain free.

## 2. System architecture

Keep the Flutter client focused on authenticated presentation and user actions. The backend is the policy enforcement point: it verifies the Firebase identity, checks the staff member's role and record-level access, validates the requested operation, and keeps privileged credentials and email-sending authority off the client.

```text
Authorized staff member
        |
        v
Flutter Admin Dashboard
  - Firebase Authentication sign-in
  - CRM views and editable email drafts
  - Explicit human approval action
        |
        | HTTPS/callable request + Firebase ID token
        v
Firebase Functions (trusted backend)
  - Verify identity, role, record access, and request
  - Read only permitted Firestore records
  - Minimize/redact context before any model request
  - Call an approved model using server-side credentials
  - Save generated content as a draft, never as an automatic send
  - Re-check authorization, opt-out, and suppression before sending
        |                         |
        v                         v
Cloud Firestore             Approved email service
  - CRM records               - Send only after approval
  - Drafts and status         - Return delivery result
  - Suppression state
```

Firebase Cloud Functions (or an equivalent trusted server) can validate AI results, send approved email, schedule follow-up work, and record send outcomes. Firestore Security Rules should prevent unauthorized client reads and writes, but they do not replace backend authorization. Privileged server SDK access may bypass Firestore Security Rules, so every Function must enforce the same authorization and validation policy itself. Do not put model-provider secrets, service-account credentials, or privileged email credentials in the Flutter bundle or source repository.

For model calls, select the current supported integration for the chosen deployment. Gemini and Firebase AI Logic are examples in the Firebase ecosystem, not a promise of availability for every account, region, workflow, or backend. Firebase AI Logic's supported client flows and protections should be checked against its current documentation; private CRM context, privileged tools, and outreach decisions should remain behind the trusted backend. Do not hardcode a live model ID; resolve a currently supported model through configuration and the provider's current documentation.

## 3. CRM agents and boundaries

Treat these as bounded workflow assistants or task types, not independent staff accounts. The orchestrator routes requests to an allowlisted capability; it does not grant an agent access just because a user asked. Each task receives only the data its authorized purpose requires.

| Agent | Possible assistance | Boundary |
| --- | --- | --- |
| **Customer Agent** | Classify customer enquiries or quote requests, summarize an authorized customer history, and prepare a service-response or quote draft | Do not invent service availability, prices, promises, or customer facts; a staff member verifies the response |
| **Partnership Agent** | Organize partnership records, assess candidate opportunities from identified sources, and draft follow-ups | Verify any discovered company/contact details; do not infer commitments or send messages; distinguish recorded facts from suggestions |
| **Community Agent** | Organize approved local community/group relationships and prepare a promotional or collaboration draft | Do not publish, contact members, or expose one person's information to another |
| **Recruiter Agent** | Help organize recruiting workflow notes, candidate relationships, or draft communications and follow-up tasks | Keep a human responsible for employment decisions; use only approved, necessary candidate information |
| **Executive Agent** | Prepare concise business summaries from records the requester is authorized to see | Prefer aggregated or minimized information; enforce role-based access and label uncertainty |
| **Email & Follow-up Agent** | Draft, revise, classify, and queue email content or a follow-up reminder | Draft and queue only; a separate server-side send action requires an authorized human approval and a fresh eligibility check |
| **Orchestrator Agent** | Route a user's request to a specific approved workflow and collect its result | No unrestricted tool access, cross-role privilege escalation, or direct bypass of review and send controls |

The agents do not each need a separately purchased model: one currently supported model may serve multiple workflows when their prompts, data access, and permissions are kept separate. Store agent instructions and output schemas under version control. Test each workflow with representative records and denied-access cases. Keep a person accountable for decisions and external communications.

An illustrative partnership workflow is: a staff member or approved research step identifies a possible partner and records the source; the Partnership Agent prepares a sourced opportunity summary; the Email & Follow-up Agent creates a tailored draft; the Orchestrator places it in the review queue; an authorized manager reviews and approves it; and the server rechecks sending eligibility, sends, and records the result. Research output is a lead to verify, not a confirmed relationship or permission to contact.

## 4. Illustrative Firestore collections and fields

The following schema is a suggested example, not a claim about an existing Cleaning with Dee database. Start with the minimum fields required for the approved workflow; use document references or IDs instead of copying personal data across collections.

| Collection | Example fields | Purpose and cautions |
| --- | --- | --- |
| `staffProfiles` | `uid`, `role`, `active`, `createdAt`, `updatedAt` | Map authenticated staff to approved application roles. Do not trust a role supplied by the client. |
| `contacts` | `name`, `email`, `organisation`, `contactType`, `source`, `status`, `notes` | Unified customer, partner, community, recruiter, and executive contact record. Store only necessary information; `source` should record where the data came from. |
| `partnerships` | `company`, `opportunity`, `contactId`, `stage`, `nextAction` | Partnership prospects and collaboration progress. Keep notes factual, relevant, and access-controlled. |
| `emailQueue` | `recipient`, `subject`, `body`, `status`, `campaignId` | Drafts and pending approvals. Client writes must not be able to trigger a send; a pending draft is not permission to send. |
| `emailLogs` | `recipient`, `sentAt`, `status`, `providerMessageId` | Restricted delivery record. Define retention and avoid duplicating message content unnecessarily. |
| `tasks` | `contactId`, `action`, `dueDate`, `result`, `owner` | Follow-up, meeting, and call tasks. |
| `activities` | `contactId`, `activityType`, `description`, `createdAt` | Contact activity history; keep descriptions purpose-limited and avoid unnecessary sensitive detail. |
| `consents` | `contactId`, `consentStatus`, `lawfulBasis`, `recordedAt` | Record permission evidence or the reviewed processing basis as appropriate. These fields are not a legal conclusion by themselves; retain the evidence/source and context needed to substantiate the record. |
| `suppressionList` | `email`, `reason`, `unsubscribeAt` | Restricted list checked before every send. Retain the minimum information needed to honor an objection or opt-out and limit access appropriately. |

The names and fields above preserve the source proposal's example schema; they are not a mandatory or production-ready schema. The exact field names and legal meaning of any permission or marketing status must be decided by the organization. Keep consent or other permission evidence where appropriate, capture its source and context, and make the suppression check authoritative at send time. Define document-level authorization, retention, correction, and deletion behavior before loading real CRM data.

## 5. Human-reviewed email workflow

1. An authenticated staff member selects an authorized contact or stakeholder record and requests a draft for a stated purpose.
2. The backend validates the user's role and record access, checks whether the request is allowed, and constructs a minimized prompt from approved CRM fields. Do not send unnecessary personal data to the model.
3. The model returns draft content. The backend validates the result and stores it in `emailQueue` with a pending-review status and a reference to the source records and prompt version.
4. A staff member reviews and edits the recipient, subject, body, purpose, and supporting facts in the dashboard. The interface must distinguish generated text from verified CRM data.
5. On explicit approval, the backend re-validates the approving user's permissions and re-checks recipient eligibility, applicable consent/permission records, the suppression list, and any recorded objection or opt-out.
6. Only if the checks pass does the backend submit the message to the approved SMTP or email provider. A Firebase Trigger Email Extension may be an option for a supported Firestore-triggered setup, but configure it so only trusted server code can create sendable records; never let a client write directly to a collection that triggers email. Record the approval identity and time, send attempt, provider result, and any suppression/block reason in a restricted `emailLogs` record without retaining unnecessary message data.
7. If eligibility is unclear, evidence is missing, a record is suppressed, or the send fails, do not send. Return the item to a human for correction or resolution.

Do not expose a client-side `sendEmail` capability that bypasses these checks. A generated draft, a user account, or an earlier eligibility check is not sufficient authorization for a later send.

## 6. UK privacy and electronic-marketing safeguards

This guide is not legal advice. Whether a message is direct marketing, which PECR rules apply, the recipient's subscriber type, the lawful basis for processing personal data, and whether an exception applies depend on the facts. Obtain organization-specific legal or privacy advice before contacting people or importing historical CRM data.

- **Check PECR before electronic outreach.** ICO guidance distinguishes corporate subscribers from individual subscribers. A limited company address may be treated differently from a sole trader or some partnership addresses, which can be treated as individual subscribers; electronic marketing to individual subscribers generally requires consent unless a narrow exception such as the customer “soft opt-in” applies. Confirm the recipient type and current conditions rather than relying on a domain name or an assumed B2B exemption. Do not assume that a publicly available address or an AI-generated message makes outreach permissible.
- **Consider UK GDPR separately.** Where personal data is used, identify and document an appropriate lawful basis and meet transparency, purpose-limitation, minimization, accuracy, security, and retention duties. PECR permission and a UK GDPR lawful basis are related but distinct checks.
- **Honor objections and opt-outs.** Keep an effective suppression mechanism and check it immediately before sending. UK GDPR gives individuals an absolute right to object to processing for direct marketing; retain only what is needed to prevent future unwanted contact, with restricted access.
- **Minimize data shared with AI providers.** Use only approved fields needed for the task. Review the provider's current terms, retention, training/data-use controls, international transfers, subprocessors, and account configuration before sending personal data.
- **Do not automate consequential decisions.** Keep human oversight over outreach, recruiting, and decisions affecting people. Provide a route to correct data and handle privacy requests.
- **Document and review.** Record the intended purpose, data sources, approvals, retention period, access roles, suppression procedure, and incident contact. Reassess when the workflow, provider, or applicable guidance changes.

Official ICO references:

- [ICO: Electronic mail marketing under PECR](https://ico.org.uk/for-organisations/direct-marketing-and-privacy-and-electronic-communications/guide-to-pecr/electronic-and-telephone-marketing/electronic-mail-marketing/)
- [ICO: Direct marketing guidance](https://ico.org.uk/for-organisations/direct-marketing-and-privacy-and-electronic-communications/direct-marketing-guidance/)
- [ICO: Business-to-business marketing](https://ico.org.uk/for-organisations/direct-marketing-and-privacy-and-electronic-communications/business-to-business-marketing/)
- [ICO: Guide to the UK GDPR lawful basis](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/lawful-basis/a-guide-to-lawful-basis/)
- [ICO: Right to object](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/individual-rights/individual-rights/right-to-object/)

## 7. Illustrative Flutter call

Add only the packages needed by the selected path. An illustrative Firebase AI Logic client setup may use `firebase_ai` and `firebase_app_check`; the CRM draft workflow below uses `cloud_functions` to keep private CRM context and privileged model access on the backend. Check the current Flutter setup guide, supported model configuration, and App Check requirements before use:

```powershell
flutter pub add cloud_functions
# Optional for a supported, non-sensitive Firebase AI Logic client flow:
flutter pub add firebase_ai firebase_app_check
```

This minimal example uses the Flutter `cloud_functions` package to ask an authenticated backend to create a draft. It intentionally sends a record reference rather than a name, email address, or free-form personal details. It is illustrative only; configure the Firebase project, callable Function, error handling, and authorization for the actual app.

```dart
import 'package:cloud_functions/cloud_functions.dart';

Future<Map<String, dynamic>> createOutreachDraft(String contactId) async {
  final callable = FirebaseFunctions.instance.httpsCallable('createOutreachDraft');
  final result = await callable.call(<String, Object?>{
    'contactId': contactId,
    'purpose': 'customer_follow_up',
  });

  final data = result.data;
  if (data is! Map) {
    throw const FormatException('Unexpected draft response.');
  }
  return Map<String, dynamic>.from(data);
}
```

The Function must verify authentication, role, and access to `contactId`; load only approved context; apply request and rate limits; and save a draft for review. A client-supplied purpose is input to validate, not permission. Keep provider and email credentials in a server-side secret manager. In production, use typed response models and handle callable errors without displaying sensitive backend details.

Firebase AI Logic can also support eligible client-side AI flows. Keep this illustrative example limited to non-sensitive text; use the trusted backend for private CRM records and outreach drafts. Supply a model identifier selected from current supported-model documentation/configuration rather than hardcoding one:

```dart
import 'package:firebase_ai/firebase_ai.dart';

Future<String> summarizeNonSensitiveText({
  required String supportedModelId,
  required String publicText,
}) async {
  final model = FirebaseAI.googleAI().generativeModel(
    model: supportedModelId,
  );
  final response = await model.generateContent([
    Content.text('Summarize this non-sensitive text: $publicText'),
  ]);
  final text = response.text;
  if (text == null) {
    throw StateError('The model returned no text.');
  }
  return text;
}
```

Configure App Check and verify current Firebase AI Logic setup, account, model, privacy, quota, and billing requirements. A client-side SDK does not authorize access to private Firestore records or email sending. If Firebase AI Logic is used directly from Flutter for an eligible flow, enable App Check and do not send private customer data or treat the client response as authorization to send an email.

## 8. Phased MVP priorities

| Phase | Deliverable | Exit check |
| --- | --- | --- |
| **0. Process and privacy readiness** | Map user roles, contact sources, current email practices, lawful-basis/PECR review, data retention, and success measures | An accountable business owner and privacy/security reviewer approve the pilot scope and test data |
| **1. Secure CRM foundation** | Flutter Admin Dashboard sign-in, role-aware views, minimal Firestore records, server-side authorization, audit metadata | Denied users cannot read or change records; rules and Functions are tested against role and record-access cases |
| **2. Read-only AI assistance** | One bounded summarization or classification workflow using approved, minimized data | Quality is reviewed on representative cases; no outbound messages or consequential decisions are automated |
| **3. Draft-only communications** | Gemini-assisted email drafts and classification for Customer and Partnership work; add other teams' drafts only when approved | Every draft shows its source and generated status; no model or client path can send it |
| **4. Controlled human-approved sending** | Admin approval, server-side sending, follow-up scheduling, suppression/eligibility checks, and send logs | End-to-end tests prove denied, opted-out, suppressed, or unapproved messages cannot be sent |
| **5. Partnership and community workflows** | Add Partnership and Community assistance, then track tasks and follow-up | Verify sources and access; no unsupervised outreach |
| **6. Recruiting and executive workflows** | Add Recruiter and Executive summaries and reminders only after privacy/access review | Employment decisions remain human-owned; executive data is role-restricted |
| **7. Orchestration and reporting** | Route bounded work among approved agents; add performance reports and analytics later | Each addition has an owner, documented access boundary, measurable benefit, fallback, and rollback plan |

Start with secure identity, data access, and unified CRM contacts. Then add AI classification/drafting and an approval-controlled send path; later add follow-up schedules, Partnership/Community, Recruiter/Executive, orchestration, and reporting in that order. Compare outcomes with the existing process and stop or revise a phase if quality, compliance, or operating cost is unacceptable.

## 9. Later expansion

After the CRM and review workflow are reliable, an approved research process could collect candidate leads such as property managers, local community organizations, or cleaning-service partners, record their sources, and prepare them for human verification before CRM entry. A later pipeline may classify verified leads, support a partnership workflow, prepare an email draft, require human approval, send through the server, and track follow-up and analytics. Discovery results must be treated as unverified until a person checks them; collection is not permission to contact.

Keep Firestore as the source of truth for CRM data. Google Drive may hold contracts and proposal documents, Notion may hold operational procedures, and Google Sheets may be used for bounded exports or analysis. Avoid keeping duplicate contact databases in these tools.

## Official references

Check current service documentation for supported features, availability, pricing, limits, and data handling. Firebase client APIs, free-tier terms, model access, and provider behavior can change; these links are starting points, not endorsements.

- [Firebase Authentication for Flutter](https://firebase.google.com/docs/auth/flutter/start)
- [Cloud Firestore security rules](https://firebase.google.com/docs/firestore/security/get-started)
- [Firebase Cloud Functions](https://firebase.google.com/docs/functions)
- [Firebase AI Logic](https://firebase.google.com/docs/ai-logic)
- [Firebase AI Logic Flutter setup](https://firebase.google.com/docs/ai-logic/get-started?api=dev)
- [Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [Gemini API pricing](https://ai.google.dev/gemini-api/docs/pricing)
- [Flutter `cloud_functions` package](https://pub.dev/packages/cloud_functions)
- [Firebase Trigger Email Extension](https://firebase.google.com/products/extensions/firebase-firestore-send-email)
