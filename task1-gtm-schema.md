# Task 01 — GTM Event Schema: OrthoNow

## A note on scope before the table

A few of these events (button clicks, page views, file link clicks) GTM can capture natively with its built-in triggers — Click, Link Click, History Change, Scroll Depth. The 3-step booking form is different: GTM has no native trigger that understands "step 2 of a multi-step form" because that state lives entirely in JavaScript/React state on the front end, not in the DOM as a distinct page or URL. GTM can only react to something — a DOM event, a URL change, a variable change — and on a JS-driven multi-step form, nothing observable changes in the DOM/URL between steps unless the front-end developer explicitly pushes a `dataLayer.push()` at each step transition. So for the booking funnel, GTM is just a listener; the instrumentation work happens in the application code. This is addressed explicitly in the funnel section below and is the basis of the brief I'd give the dev team.

## Full Event Schema

| Event Name | Trigger Type | Key Parameters (min. 3) | Feeds Into (GA4) |
|---|---|---|---|
| `booking_step_complete` | Custom Event (dataLayer push, fired by front-end on each step transition) | `step_number`, `step_name`, `clinic_location`, `specialty` (step 1 only) | Funnel Exploration report, "Booking Funnel" audience for retargeting |
| `booking_confirmed` | Custom Event (dataLayer push on step 3 success / booking API 200 response) | `clinic_location`, `specialty`, `preferred_date`, `booking_id` | Conversion event, Key Events report, "Converted Leads" audience, Google Ads import |
| `call_now_click` | Click Trigger (Just Links / All Elements, filtered on `tel:` href or `data-event="call-now"` attribute) | `page_location`, `click_text`, `clinic_location` (data attribute on the page) | Engagement report, phone-lead audience for offline conversion matching |
| `whatsapp_chat_open` | Click Trigger (filtered on element matching the WhatsApp widget's button ID/class, or href containing `wa.me`) | `page_location`, `widget_state` (open/close), `clinic_location` | Engagement report, "Chat Initiated" audience |
| `patient_guide_download_form_submit` | Custom Event (Form Submission trigger on the gated form, or dataLayer push on success callback) | `form_name`, `page_location`, `lead_source` | Lead Gen report, "Content Downloaders" audience for nurture campaigns |
| `patient_guide_download_pdf_open` | Click Trigger (Just Links, filtered on `.pdf` href, fires only after gate is passed) | `file_name`, `file_url`, `page_location` | File download report, content engagement |
| `clinic_page_view` | Trigger: History Change / Page View, filtered on URL path pattern `/clinics/*` | `clinic_name`, `clinic_city`, `page_location` | Pages and Screens report, per-clinic engagement, location-based remarketing audiences |
| `blog_scroll_depth` | Scroll Depth trigger (GTM built-in), thresholds at 25/50/75/90% | `scroll_percentage`, `article_title`, `page_location` | Engagement report, "Engaged Readers" audience for content remarketing |
| `blog_read_complete` | Custom Event, fires once `blog_scroll_depth` hits 90% AND time-on-page > 60s (combined trigger via a custom JS variable) | `article_title`, `time_on_page`, `page_location` | Content performance report, qualifies users for blog-to-booking remarketing |

Notes on triggers I'd actually configure in GTM (not just label):
- **Call Now / WhatsApp clicks** use a Click trigger with a Click Element condition (CSS selector or `data-*` attribute), not a generic "Click — All Elements" left unfiltered, otherwise every click on the page would log noise.
- **Patient Guide form**: GTM's built-in Form Submission trigger only fires reliably on traditional form submits. If the gate form submits via `fetch()`/AJAX (likely, since it needs to unlock a PDF without a page reload), GTM can't see that natively either — same problem as the booking form, smaller scale. I'd ask the front-end dev to push a `dataLayer.push({event: 'patient_guide_download_form_submit', ...})` on the AJAX success callback rather than relying on GTM's form trigger.
- **Scroll Depth** is GTM's one native trigger that handles a "multi-step-feeling" interaction without custom code — it tracks percentage scrolled per page load automatically.

## Booking Funnel: Step-Level Drop-off Tracking

### Who writes the dataLayer push?
The front-end developer does — not GTM, and not me from inside GTM. GTM can only listen for an event that's already been pushed into `window.dataLayer`; it can't detect that a React/JS state variable changed from `step: 1` to `step: 2` unless the app explicitly tells it to. My job is to specify exactly what to push and when; their job is to call it at the right point in the component lifecycle.

### What I'd brief the dev team
For each step transition (i.e., when the user clicks "Next" / "Confirm" and validation passes), the app should push a `dataLayer.push()` *before* or immediately after the UI transitions to the next step — not on render, since a step could re-render without the user actually progressing.

**Step 1 — clinic + specialty selected:**
```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}"
}
```

**Step 2 — contact details entered:**
```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_details_entered",
  "clinic_location": "{{clinic name}}",
  "preferred_date": "{{preferred date selected}}"
}
```

**Step 3 — booking confirmed:**
```json
{
  "event": "booking_confirmed",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}",
  "booking_id": "{{booking id from API response}}"
}
```

I'd also ask for a push on step *abandonment* if feasible — e.g., a `beforeunload` listener or a timeout-based push if the user leaves the tab mid-step — since drop-off between steps is invisible if we only ever log forward progress. This is a stretch ask, not a blocker for v1.

### GTM configuration
1. Create a **Custom Event trigger** for each `event` value: `booking_step_complete` (fires on all three steps, differentiated by `step_number`) and `booking_confirmed`.
2. Create **Data Layer Variables** in GTM for `step_number`, `step_name`, `clinic_location`, `specialty`, `preferred_date`, `booking_id` so they can be attached as event parameters.
3. Create a **GA4 Event tag** that fires on the `booking_step_complete` trigger, sends event name dynamically as `booking_step_complete`, and maps the dataLayer variables as event parameters. Same pattern for `booking_confirmed`, marked as a Key Event in GA4.

### Surfacing drop-off in GA4 Funnel Exploration
In GA4, I'd build a **Funnel Exploration** with three open steps:
1. Step 1: `booking_step_complete` where `step_number = 1`
2. Step 2: `booking_step_complete` where `step_number = 2`
3. Step 3: `booking_confirmed`

I'd set the funnel to "open" (not strict ordering required, since some users may navigate non-linearly) but enable "Show elapsed time" to see how long users sit on each step before dropping. GA4 will then show the percentage of users who completed each step vs. fell off, broken down further by `clinic_location` or `specialty` as a secondary dimension — useful for spotting if drop-off concentrates around a specific clinic's availability or a specific specialty's date picker.

## Conversion Action to Import into Google Ads

**`booking_confirmed`** — not `consultation_form_submitted`-style top-of-funnel events, and not `patient_guide_download_form_submit`.

Why this one: it's the only event in the schema that represents a completed appointment booking with verifiable downstream value (a `booking_id` tied to an actual clinic slot), not just an expression of interest. Importing `call_now_click` or `whatsapp_chat_open` as the Google Ads conversion would let Smart Bidding optimize toward cheap, high-volume "intent" signals that don't reliably convert to revenue — clicking a WhatsApp button costs the user nothing and many will never book. `booking_confirmed` is the closest proxy to actual patient acquisition in the available event set, so it's the one signal worth letting the bidding algorithm chase aggressively. Once there's enough call-tracking data (e.g., via a call-tracking number swap), I'd consider a secondary, lower-value conversion action for `call_now_click` to feed Smart Bidding more volume for ML training, but `booking_confirmed` should be the primary one from day one.
