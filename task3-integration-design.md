# Task 03 — Integration Design: Landing Page → HubSpot → WhatsApp → Google Ads

## Architecture

I'd skip HubSpot's native embedded form and call the **HubSpot Forms API directly** from the landing page's submit handler. Native embed means HubSpot owns the form markup and styling, which conflicts with the custom-built, conversion-optimized page in Task 2; a Forms API POST keeps full control of UX while still landing the submission natively in HubSpot's contact timeline (better than routing through Zapier/Make for the primary write, since those add a hop, a few seconds of latency, and a paid-plan dependency for something a single `fetch()` call handles directly).

Flow: form submit → client-side validation → `fetch()` POST to HubSpot Forms API with `name`, `phone`, `clinic_preference`, plus hidden fields for `lifecyclestage`/`lead status = New Enquiry` and `hs_lead_source = Google Ads - Consultation Landing Page` → on the same submit, fire the `dataLayer.push({event: 'consultation_form_submitted'})`, which a GTM tag turns into a server-side or client-side Google Ads conversion hit, decoupled from whether HubSpot's call succeeds. In parallel, I'd use a **HubSpot Workflow** (trigger: contact property `lead status = New Enquiry`) to call a webhook that hits Namoza's Karix WhatsApp Business API endpoint with the templated confirmation message. Workflow-as-orchestrator beats wiring Karix directly off the form POST, because the workflow gives retry logic and a visible execution log for free.

## Biggest failure point: phone number deduplication

HubSpot deduplicates contacts on **email by default**, not phone — and this form collects no email. Two patients submitting with the same phone number but different names will create **two separate contacts**, not an update to one, unless I explicitly configure phone as a unique identifier. The fix: set `phone` as a required, unique contact property in HubSpot and use the Forms API's update-by-property behavior (or a pre-check via the Contacts Search API keyed on phone) before creating a new contact, so a repeat submission updates the existing record's `clinic_preference` and `lead status` rather than forking the patient's history into duplicate timelines.

## Monitoring the 2-minute WhatsApp SLA

Failure points: Karix API downtime, workflow execution delay, or a malformed phone number blocking template delivery. I'd monitor via a HubSpot workflow branch that flags any contact still in "WhatsApp not sent" status after 2 minutes, alerting the team in Slack, with a fallback SMS via the same Karix account if WhatsApp delivery fails.
