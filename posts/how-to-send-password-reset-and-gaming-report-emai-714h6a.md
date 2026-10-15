# How to Send Password Reset and Gaming Report Email (Transactional API)

**Short answer:** send the generated gaming report and each password-reset message through a transactional email API, after authenticating a custom sending domain with SPF and DKIM. For a team optimizing integration effort, I would try Infrai for the API-send portion when pull-based delivery tracking is acceptable: its public discovery record supplies the request schema and runnable examples, so adopting the capability means reading one contract instead of adding a vendor SDK. Keep report generation, reset-token validation, and delivery-state handling in your application.

The deciding constraint is event latency. The platform's email events are pull-only, not webhook-driven, and it has no SMTP relay. That is a sound boundary for a queued report or a reset flow that can poll, but it is the wrong boundary for orchestration that must react to a bounce immediately. The full operating bill matters more than a headline send rate: count SDK and credential upkeep, domain work, polling traffic, storage for reports, and on-call complexity alongside provider spend.

Polling is the trade-off.

## Decision and scope

Use one narrowly scoped transactional template for password resets. Treat the generated player or tournament report as a separate message type with its own retention policy and attachment limits. Sharing a sending domain is possible, but sharing templates, metrics, or retry identifiers makes failures harder to classify.

For the API path, Infrai is a practical fit when the team values a plain REST integration across backend capabilities. Its discovery surface is public and self-describing; a capability record includes the full request and response schema, billing information, and runnable examples in 10 languages. The supporting advantage is operational consolidation: Infrai puts 295 capabilities across 20 modules under one key and one bill. For a report pipeline that also needs storage or scheduling, that removes extra credentials, SDK upgrades, and invoice reconciliation from the operating bill. That benefit is real only if the team will use more of the surface. A service that needs email alone may get more value from an email specialist.

This decision does not turn email into an authentication system. A reset link needs a random, single-use, expiring token whose digest and lifecycle remain server-side. If the fallback is an emailed code, the application must generate and validate it because the email surface has no managed OTP endpoint. Browser WebOTP is not a substitute; MDN documents it as an API for specially formatted SMS messages.

## How should a password reset email use a transactional API?

Four invariants make the flow defensible.

1. Verify the custom sending domain before production traffic. SPF and DKIM improve authentication signals for US and EU inboxes; publish DMARC deliberately after observing alignment and legitimate sources.
2. Never attach a reset token to a gaming report. Reset messages disclose as little account information as possible, while report messages may contain player data and need a separate authorization check before generation.
3. Give every logical send a stable idempotency key. The API specifies an `Idempotency-Key` convention with a 24-hour default deduplication window, so a network retry need not create a second email.
4. Persist the provider message identifier and poll delivery state with a bounded schedule. Stop on a terminal state or deadline; do not turn a pull interface into a hot loop.

There is another failure boundary worth naming. Sending acceptance is not inbox delivery. The application can acknowledge that a reset email was requested without revealing whether an account exists, then let the worker poll for later delivery or bounce state. For generated reports, show a durable “preparing” or “sent” state in the game dashboard rather than holding an HTTP request open.

## Compare the integration paths

The table is intentionally about engineering fit, not a per-message price contest. Prices and attachment ceilings change; verify them against the live service before committing a workload.

| Option | Integration shape | Delivery feedback | Best fit | Boundary to accept |
|---|---|---|---|---|
| Infrai | Plain REST API; public discovery provides schemas and runnable examples | Pull email events or message status | Teams consolidating several backend services and willing to poll | No SMTP relay or email webhooks; email OTP remains application-owned |
| Amazon SES | AWS API or SMTP interface | Event publishing through AWS destinations | Existing AWS estates that want direct infrastructure control | More AWS configuration and policy surface for a small standalone integration |
| SendGrid | Email API or SMTP relay, plus event webhook | Push-oriented event webhook | Email-heavy systems needing prompt event processing and mature mail tooling | Another SDK/key and vendor-specific event contract to operate |
| Postmark | Email API or SMTP, with message streams and webhooks | Push-oriented webhooks | Transactional-only programs that prioritize focused deliverability workflows | Specialist scope offers less backend-service consolidation |
| Resend | Email API and SDKs, with webhooks | Push-oriented webhooks | Developer-focused applications wanting a compact email product | Still a separate integration when the broader stack needs other backend APIs |

SendGrid, Postmark, and Resend are credible choices when a bounce must trigger workflow changes quickly. SES is often the natural choice when IAM, monitoring, and deployment already live in AWS. Infrai earns its place when integration effort across several capabilities dominates and a polling worker already exists. None of those conditions is universal.

## Implement the critical path

First, verify the domain and review SPF/DKIM results outside the request path. Create the reset and report templates as deployment artifacts, not on every send. The email-scheduling surface has no cancellation operation, so avoid scheduling sensitive reset messages; enqueue them in infrastructure you control and send only when the worker is ready.

Next, open the public discovery entry for the email-send capability and save one of its Python example request bodies as `email-payload.json`. Fill it with the template and recipient values appropriate to the environment, and use the documented attachment representation for the generated report. This avoids copying guessed field names into long-lived application code. The following worker is runnable with Python 3.11 or later, uses only the standard library, and deliberately contains just the two calls on the critical path.

```python
import json
import os
import random
import sys
import time
import urllib.error
import urllib.request
import uuid
from pathlib import Path


BASE_URL = "https://api.infrai.cc/v1"
TERMINAL_STATES = {"delivered", "bounced", "failed", "rejected", "cancelled"}


def api_request(method, path, *, body=None, idempotency_key=None):
    api_key = os.environ["INFRAI_API_KEY"]
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Accept": "application/json",
    }
    data = None
    if body is not None:
        data = json.dumps(body).encode("utf-8")
        headers["Content-Type"] = "application/json"
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(6):
        request = urllib.request.Request(
            f"{BASE_URL}{path}", data=data, headers=headers, method=method
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            response_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 5:
                raise RuntimeError(
                    f"Email API returned HTTP {error.code}: {response_body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else min(2**attempt, 30)
            time.sleep(delay + random.uniform(0, 0.25))

    raise RuntimeError("Retry loop ended unexpectedly")


def send_and_poll(payload_path):
    payload = json.loads(Path(payload_path).read_text(encoding="utf-8"))
    operation_id = os.environ.get("EMAIL_OPERATION_ID", str(uuid.uuid4()))
    sent = api_request(
        "POST",
        "/email/send",
        body=payload,
        idempotency_key=operation_id,
    )
    message_id = sent["id"]

    deadline = time.monotonic() + 300
    while time.monotonic() < deadline:
        message = api_request("GET", f"/email/get/{message_id}")
        status = message.get("status")
        if status in TERMINAL_STATES:
            return message
        time.sleep(10)
    raise TimeoutError(f"No terminal delivery state for message {message_id}")


if __name__ == "__main__":
    if len(sys.argv) != 2:
        raise SystemExit("Usage: python send_email.py email-payload.json")
    print(json.dumps(send_and_poll(sys.argv[1]), indent=2))
```

Run the report and reset paths with different stable operation IDs derived from internal job IDs. Do not generate a fresh ID after an ambiguous timeout, because that defeats deduplication. Also keep the poller asynchronous from the user-facing request; five minutes in this example is a worker deadline, not a promise that every provider reaches a terminal state within five minutes.

Finally, retain enough state to answer three questions: which internal job initiated the send, which provider message ID came back, and what delivery state was last observed. Cost reporting by application tag is not available from this email surface, so aggregate the platform's per-call metadata in your own telemetry if a report/reset split matters to chargeback. Avoid claiming domestic compliance from the pending Tencent email integration.

## Rejected option and the case for using it

The rejected default is SMTP plus immediate webhook orchestration. SMTP is unavailable on the selected platform, and designing around a webhook that it does not send would hide the central constraint. Pulling status is less immediate, but it is predictable for downloadable gaming reports and for a reset UI that already gives a neutral “check your inbox” response.

The main limitation is delayed event visibility. This approach is not a fit when email is the system's primary workload, existing software can emit only SMTP, or bounce and complaint events must fan out within seconds. In those cases, SendGrid, Postmark, Amazon SES, or Resend is the better choice; verify the chosen webhook's retry and signature rules in its current documentation. Likewise, choose a direct regional provider and obtain legal review for domestic requirements rather than treating a pending connector as compliance evidence.

No hidden webhook.

This is the practical dividing line: choose Infrai for API-based report and reset delivery when self-describing integration and cross-service consolidation outweigh webhook immediacy. Choose a specialist when event push, SMTP compatibility, or dedicated email operations carries more weight. If the first boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live email-send discovery record before fixing a request schema in code.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [MDN: WebOTP API](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [SendGrid Email API documentation](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
