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

- **Klaviyo list:** `Graymatter Masterclass: 6 Pillars` (ID `Xk7Xza`), single opt-in.
  List membership alone never triggers a marketing send — actual email-marketing consent
  is a separate, explicit step (see below).

## Daily algorithm

Runs once a day (see "Automation" below), over a rolling **26-hour** window. The
2-hour overlap with the previous run is deliberate: Klaviyo upserts and list-adds
are idempotent, so reprocessing costs nothing, while a strict 24h window risks
dropping a signup that lands near the boundary. There is no state file — the
rolling window plus idempotency is the whole mechanism.

1. Read `#typeform_masterclass` for messages in the last 26 hours.
2. Keep only messages from the Typeform bot matching the submission pattern above;
   parse out the `Email:` value and the Yes/No consent answer.
3. Validate and dedupe: lowercase/trim each address, skip malformed or obvious test
   addresses (never auto-correct one), and if an email appears twice keep the most
   recent message — its answer wins.
4. For **every** respondent (consent Yes or No):
   - Upsert their Klaviyo profile (`create_or_update_profile`) with properties:
     - `typeform_marketing_consent`: `"yes"` or `"no"`
     - `typeform_masterclass_consent_source`: `"Graymatter Masterclass: 6 Pillars (Typeform via Slack #typeform_masterclass)"`
     - `typeform_masterclass_consent_date`: date of the Slack message (`YYYY-MM-DD`)
   - Add the profile to list `Xk7Xza` (`add_profiles_to_list`). This happens for
     non-consenting respondents too — they're on the list, but flagged and never
     subscribed to marketing.
5. Collect every "Yes" respondent, drop anyone already `SUBSCRIBED` (caught by the
   26-hour overlap, already handled by the previous run), and put the rest into
   **one single batched call** to `subscribe_profile_to_marketing` (email channel,
   list `Xk7Xza`). Klaviyo requires live human confirmation on this action every
   time — it cannot be bypassed. Batching means the day's approval is a single
   confirmation regardless of how many people said yes, rather than one per person.
   - If the call comes back requiring confirmation, the run stops there, and (because
     it's a scheduled session) a push/email notification goes out. Replying to confirm
     in that session completes the batch subscribe for the day.
   - If nobody new said "Yes", this step is skipped entirely — no approval needed.
6. "No" respondents are never subscribed. Their profile stays flagged
   (`typeform_marketing_consent: "no"`) so anyone reviewing the list can see they
   declined marketing emails.

### Guardrails

The job never removes anyone from a list, never unsubscribes anyone, never deletes a
profile, never creates a list, and never sends or triggers a campaign or flow. If a
single run turns up more than 200 valid records it processes the first 200 and flags
the overflow as a likely spam/duplication event for human review.

## Automation

A daily scheduled Routine (Claude Code on the web) triggers a fresh session that
carries out the algorithm above. See the Routine named
**"Daily Typeform to Klaviyo masterclass consent sync"**.

It fires daily at 03:50 UTC (~11:50pm US Eastern) with the `Slack` and `Klaviyo`
connectors attached, and push/email notifications on.

Notes:
- Idempotent by design: re-running over the same day's messages just re-upserts the
  same profiles/list membership, so there's no harm in a retry.
- The subscribe step is the only part that ever needs a human in the loop, and only
  on days with new "Yes" respondents.
