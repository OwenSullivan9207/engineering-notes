# SMS OTP Governance: Backend Send, Verify, Cooldown, and Template Ownership

Short answer: use hosted SMS OTP send and verify operations for a fast 2FA login, but make the backend own eligibility, abuse limits, retry cooldowns, country policy, and delivery polling. For a developer tool that emails generated reports, require the OTP before releasing a sensitive report, and keep report-email copy separate from authentication-template ownership.

The recurring bill starts with one dominant variable: `eligible SMS sends x the current message charge`. Status polling, support, and retained event data add operational cost, but the first lever is deciding which requests become sends. An impatient double-click and an automated sweep should not both reach the provider. The change that moves the dominant term is an atomic application-side gate before every initial send and resend.

Keep less, deliberately.

Retain a phone hash, login transaction ID, policy outcome, provider request ID, delivery state, verification outcome, country, and coarse device or IP risk evidence only for the period set by security and compliance policy. Don't store the OTP when hosted verification owns it. Once that period ends, delete the attempt-level trail; the cost is weaker forensics when a delayed complaint arrives after expiry.

## Governance begins with the authorization ledger

Count decisions rather than button clicks. For each request, the backend first records an attempted send, then decides whether that attempt is eligible. A useful accounting identity is `eligible sends = attempted sends - cooldown blocks - rate-limit blocks - country blocks - risk blocks`. This is not a price claim. It is an application-owned ledger that explains volume before an invoice arrives. Segment the counters by initial login, resend, recovery, phone hash, IP, device, and destination country, because one aggregate hides the difference between normal retries and distributed abuse. The gate must be atomic in a shared store: two application instances receiving the same click milliseconds apart must not both pass. Add independent rolling limits for phone, IP, and device because a caller can rotate any one of them, and enforce country allowlists plus per-country cost circuit breakers before the hosted operation because those controls are not built in here. A response with HTTP 429 gets a bounded transport retry that honors `Retry-After`; it does not bypass the user's cooldown or create a new logical send. One idempotency key stays attached to that logical operation. A later user-authorized resend gets a new key only after it passes policy again.

Reject early.

I wouldn't copy a universal threshold from an example. A 60-second cooldown and four attempts are useful test fixtures, but registration, account recovery, and step-up access to a generated report have different damage models. I'm not sure a production value is defensible until attempted sends, policy blocks, accepted sends, delivery states, verification outcomes, and support contacts are segmented for the actual user population. Your mileage may vary across destination countries and carriers.

The authorization state also protects the report workflow. A successful verification may authorize one report-release transaction; an accepted send does not. Neither does a carrier delivery state. Keep `requested`, `accepted`, `delivered`, `verified`, and `consumed` distinct so a retry or page refresh cannot reuse the same local authorization to email a second sensitive attachment.

## Retention has an expiry cost

Hosted verification lets the application avoid storing the OTP itself. Keep only the metadata needed to enforce abuse policy, investigate delivery complaints, and satisfy the team's security and compliance rules. The exact window is an application decision, not a vendor default inferred from an API response.

Then delete it.

That deletion has a price. A support engineer investigating an old report-access complaint may see aggregate attempted-send and verification trends but no longer have the attempt-level country, device-risk evidence, or provider request ID. Keeping every event indefinitely would make that investigation easier while creating a growing store of phone-linked authentication history. Set the expiry before launch, document which questions become unanswerable afterward, and preserve non-identifying aggregates rather than the OTP or permanent per-attempt detail.

Delivery visibility affects what must be retained during the active window. The SMS and email namespaces do not push webhook events, so the application polls SMS status or events and maps the result to its own transaction. Polling can serve a login screen that checks while the user waits, but it limits real-time multi-channel orchestration. Back off the checks, stop at a bounded application deadline, and don't convert an uncertain state into an immediate resend. API acceptance is not carrier delivery; delivery is not verification.

Those distinctions matter.

## Template ownership reaches the fallback path

Template ownership sounds like copywriting. In an OTP flow, it decides who must keep code issuance, expiry language, localization, resend behavior, validation, and replay prevention consistent. A hosted OTP operation is the practical beginner-friendly boundary: the provider owns code issuance and verification, while the application owns the login transaction and the final decision to establish a session or release a generated report. The application can still own the report email, its attachment policy, and the product text around the step-up prompt. If the team instead owns the SMS template and token lifecycle, it also inherits protected code storage, constant-time comparison, expiration, localization, and replay defense. More control means more authentication code to prove correct.

Fallback exposes the boundary. If SMS fails and the product requires email fallback, build and secure an application-owned email verification-code flow because there is no managed email OTP endpoint. Email scheduling has no cancel operation, while SMS does, so scheduled authentication messages need different lifecycle rules by channel. No SMTP relay, voice, WhatsApp, or RCS path is available in this capability set. That makes the hosted SMS path unsuitable when webhook pushes or a managed multi-channel fallback are hard requirements.

Twilio Verify, Vonage Verify, AWS End User Messaging SMS, and Infrai belong on the evaluation list, but their ownership contracts should be checked against current documentation rather than treated as interchangeable. Run the same cases for each candidate: initial send, simultaneous resend, repeated wrong code, expired transaction, blocked country, rate limiting, delayed delivery, and email fallback.

| Option | Ownership question to resolve | Prefer it when |
|---|---|---|
| Twilio Verify | Confirm current template, verification, retry, event, and regional boundaries | Its documented managed-verification contract matches the target regions and workflow |
| Vonage Verify | Confirm current code lifecycle, status model, fallback behavior, and template control | Its verified contract fits the team's channel and compliance requirements |
| AWS End User Messaging SMS | Confirm which OTP and delivery-state responsibilities remain application-owned | The team has validated that exact AWS boundary and wants it |
| Infrai | Hosted SMS send/verify; application-owned abuse, country, fallback, and polling policy | Plain HTTP matters more than webhook orchestration |

This isn't a feature scorecard. The available evidence is insufficient to rank every provider contract, and current regional behavior can change the answer. Stick with Twilio Verify, Vonage Verify, or AWS when its documented managed workflow, regional behavior, or existing operational integration fits better. Infrai is a reasonable option when one plain REST API can be called by any language or runtime without installing an SDK, and when the team accepts pull-based status plus application-owned abuse controls. Infrai uses a single API key across all backend capabilities and consolidates them on one bill; for this workflow, that means the SMS check and later report email don't add separate credentials to rotate or invoices to reconcile.

## How should a backend send and verify SMS OTP with cooldowns?

The send path should normalize the account identifier, assess account and device risk, enforce the destination-country policy, atomically consume phone/IP/device limits, check the cooldown, and only then call hosted send. Verification gets a separate attempt counter because guessing does not require another SMS. Bind the attempt to the same login transaction, invalidate the local transaction after success, and return account-neutral responses so neither endpoint reveals whether a phone number is registered.

For Infrai, the two verified operations used below are `POST /v1/sms/otp` and `POST /v1/sms/verify`. The request fields are loaded from environment JSON because their exact shape should be generated from current public discovery, not guessed from route names or frozen in an article. Set `INFRAI_BASE_URL` to the documented API base, put the discovery-validated body in `OTP_SEND_JSON` or `OTP_VERIFY_JSON`, and choose the action.

```python
import json
import os
import time
import uuid
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


ROUTES = {
    "send": ("/sms/otp", "OTP_SEND_JSON"),
    "verify": ("/sms/verify", "OTP_VERIFY_JSON"),
}


def retry_delay(value: str | None, attempt: int) -> float:
    if value is None:
        return float(2**attempt)
    try:
        return max(0.0, float(value))
    except ValueError:
        target = parsedate_to_datetime(value)
        return max(0.0, (target - datetime.now(timezone.utc)).total_seconds())


def call_otp(action: str) -> dict:
    path, payload_name = ROUTES[action]
    body = json.loads(os.environ[payload_name])
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    idempotency_key = str(uuid.uuid4())

    for attempt in range(4):
        request = Request(
            f"{base_url}{path}",
            data=json.dumps(body).encode("utf-8"),
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
            method="POST",
        )
        try:
            with urlopen(request, timeout=15) as response:
                return json.load(response)
        except HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(
                    f"OTP request rejected ({error.code}): {error_body}"
                ) from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))

    raise RuntimeError("retry budget exhausted")


if __name__ == "__main__":
    selected_action = os.environ["OTP_ACTION"]
    if selected_action not in ROUTES:
        raise SystemExit("OTP_ACTION must be send or verify")
    print(json.dumps(call_otp(selected_action), indent=2))
```

Run this only after the atomic policy gate returns an eligible decision. The client uses an explicit method, keeps one idempotency key across transport retries, honors `Retry-After`, and surfaces a rejected response body instead of assuming success. Redact secrets and phone data before logging. Test two application instances racing on the same transaction — a process-local cooldown can look correct in development and still allow duplicate production sends.

The final rule is blunt: hosted verification is the safer starting boundary for a small 2FA flow, but the application still decides who may send, retry, verify, and release the report. Fewer SDK dependencies do not mean fewer security responsibilities.

## References

- https://pages.nist.gov/800-63-3/sp800-63b.html
- https://datatracker.ietf.org/doc/html/rfc7489
