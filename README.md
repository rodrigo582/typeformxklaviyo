# Typeform → Klaviyo Masterclass Consent Sync

Daily sync of *Graymatter Masterclass: 6 Pillars* Typeform responses (posted into Slack)
into a dedicated Klaviyo list, respecting each respondent's marketing-email consent answer.

## Data sources

- **Slack channel:** `#typeform_masterclass` (ID `C0C1YL5MYDC`) in the graymatterorg workspace.
  Typeform posts one message per form submission here, formatted like:

  ```
  Hey, your *Graymatter Masterclass: 6 Pillars* typeform got a new response.
  *Email:*
  <email address>
  *Do you consent to receive marketing emails from Graymatter? You can unsubscribe at any time.*
  Yes | No
  ```

- **Klaviyo list:** `Graymatter Masterclass: 6 Pillars` (ID `RAXTpK`), single opt-in.
  List membership alone never triggers a marketing send — actual email-marketing consent
  is a separate, explicit step (see below).

## Daily algorithm

Runs once a day (see "Automation" below). For the current calendar day
(America/New_York):

1. Read `#typeform_masterclass` for messages posted today, oldest → latest.
2. Keep only messages from the Typeform bot matching the submission pattern above;
   parse out the `Email:` value and the Yes/No consent answer.
3. For **every** respondent (consent Yes or No):
   - Upsert their Klaviyo profile (`create_or_update_profile`) with properties:
     - `typeform_marketing_consent`: `"yes"` or `"no"`
     - `typeform_masterclass_consent_source`: `"Graymatter Masterclass: 6 Pillars (Typeform via Slack #typeform_masterclass)"`
     - `typeform_masterclass_consent_date`: date of the Slack message (`YYYY-MM-DD`)
   - Add the profile to list `RAXTpK` (`add_profiles_to_list`). This happens for
     non-consenting respondents too — they're on the list, but flagged and never
     subscribed to marketing.
4. Collect every "Yes" respondent from the day into **one single batched call** to
   `subscribe_profile_to_marketing` (email channel, list `RAXTpK`). Klaviyo requires
   live human confirmation on this action every time — it cannot be bypassed. Batching
   means the day's approval is a single confirmation regardless of how many people
   said yes that day, rather than one per person.
   - If the call comes back requiring confirmation, the run stops there, and (because
     it's a scheduled session) a push/email notification goes out. Replying to confirm
     in that session completes the batch subscribe for the day.
   - If nobody said "Yes" that day, this step is skipped entirely — no approval needed.
5. "No" respondents are never subscribed. Their profile stays flagged
   (`typeform_marketing_consent: "no"`) so anyone reviewing the list can see they
   declined marketing emails.

## Automation

A daily scheduled Routine (Claude Code on the web) triggers a fresh session that
carries out the algorithm above, with `Slack` and `Klaviyo` connector access. See
the Routine named **"Daily Typeform → Klaviyo masterclass consent sync"**.

Notes:
- Idempotent by design: re-running over the same day's messages just re-upserts the
  same profiles/list membership, so there's no harm in a retry.
- The subscribe step is the only part that ever needs a human in the loop, and only
  on days with new "Yes" respondents.
