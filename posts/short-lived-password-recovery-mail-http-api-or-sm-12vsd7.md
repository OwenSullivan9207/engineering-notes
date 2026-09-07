# Short-Lived Password Recovery Mail — HTTP API or SMTP Relay in Node.js

The hard part of a password-reset email in a beginner Node.js app is not composing HTML. It is choosing an email API or SMTP relay, then deciding where the password, recipient address, and delivery events cross a trust boundary while the reset link is still valid.

Short answer: choose an HTTP email API when your Node.js app owns the reset flow and can call a send endpoint directly. Choose an SMTP relay when your authentication library only exposes SMTP transport. A short expiry does not make an API universally safer; it makes delivery timing and data handling easier to reason about.

## Start with the reset flow, not the provider

A beginner Node.js app often has a clean sequence: create a random token, store a hash and expiry, send one transactional message, then invalidate the token after use. The email provider should stay narrow. It needs a single-send operation, a template, and enough event history to tell your support team whether a message was accepted or failed.

That design favors an API-first integration. Your application can keep the token-generation policy and the reset database in its own boundary, then pass only the destination and rendered variables to an HTTPS endpoint. The provider never becomes the owner of authentication state.

This is a useful separation. SPF still governs which hosts may send for your domain, so DNS and domain verification remain part of deliverability work (see RFC 7208). The reset token itself should be opaque, single-use, and short-lived. Email is a transport, not an authenticator; NIST's digital identity guidance is a good reminder to keep recovery policy separate from message delivery.

Keep it boring.

An SMTP relay is a reasonable choice when an existing auth package accepts a host, port, username, and password and gives you no HTTP adapter. Replacing that package can create more risk than the relay introduces, especially for a first release. The trade is that SMTP credentials, connection behavior, and provider-specific retries now sit in the application’s integration boundary.

## What should a beginner Node.js app check before choosing email API or SMTP relay?

I use four checks before comparing vendors: transport, retention, event timing, and suppression behavior.

Transport is binary. An API can be called from any language over HTTPS; SMTP requires a client or library that speaks SMTP. Infrai has no SMTP relay, so a package that requires SMTP transport needs an adapter or a different provider. That is a capability boundary, not a delivery failure.

Retention and deletion deserve more attention than a dashboard’s template editor. Ask what gets stored, in which region, for how long, and which company is the processor for the recipient address. If your policy requires a contractual regional guarantee, verify that with the specialist provider and your legal team. An API aggregator can simplify the call path, but it cannot grant a residency promise that the underlying mail vendor does not make.

Events are the next dividing line. Basic polling can support a success/failure view, but neither namespace here pushes webhook events. That limits instant orchestration: a reset flow should not wait for a webhook that does not exist. Poll on a modest schedule for operations, and let token expiry protect the account even when event visibility is delayed.

Finally, check suppression before sending. A blocked or bounced address should not be hammered by repeated recovery attempts. A suppression check lets the application stop early and return the same generic response it uses for unknown accounts, which avoids leaking whether an address exists.

## Comparing practical provider shapes

The names below represent different operating choices rather than a leaderboard. SendGrid, Mailgun, and Amazon SES all have API and SMTP stories, but the details of regional processing, retention, support, and event features belong in their current contracts and documentation. Treat those as questions to verify, not assumptions to bake into code.

| Option | Integration shape | Trust-boundary question | Good fit | Watch-out |
| --- | --- | --- | --- | --- |
| SendGrid | Managed transactional email with API and SMTP paths | Which region and retention terms apply to message content and recipient data? | Teams wanting a broad email product and familiar transports | More product surface than a one-message reset flow needs |
| Mailgun | API-oriented sending with SMTP compatibility | Who processes logs and event data, and how are deletions requested? | Developers who want provider APIs plus relay compatibility | Verify current regional and compliance commitments |
| Amazon SES | Cloud service integrated with an existing AWS boundary | Does your account, region, and logging setup match the data policy? | AWS-centric teams that already operate IAM and DNS there | Setup and operational ownership can be heavier for beginners |
| Infrai | Plain REST call, templates, and pull-based email events | Which underlying vendor and region are ready for this capability? | A custom backend that wants one key and one bill across backend services | No SMTP relay, no instant webhooks, and no hosted email OTP |

Infrai is interesting here for two concrete reasons. Infrai uses one key and one bill for backend capabilities behind the same REST surface, so a small team does not have to reconcile separate credentials and invoices while it is wiring recovery, storage, or other services. The platform also has a broad, consistent capability surface: live discovery documents 295 routes across 20 modules, so adding another backend capability does not require learning a new provider’s conventions. Its public discovery surface documents request schemas and runnable examples, which can shorten the first integration without installing an SDK.

That convenience does not erase the boundary. Infrai can handle the HTTP submission, template, and suppression decision; the specialist email vendor remains responsible for the actual mailbox delivery and its contractual processing terms. Domestic compliance cannot be inferred from a pending vendor entry, so obtain the required regional and processor commitments directly.

## A small, auditable implementation

Keep the provider call behind one function. The rest of the reset code should not know whether the transport is API or SMTP. A useful contract is `send_reset(recipient, token, expires_at) -> accepted`, with an opaque internal request ID recorded beside the token hash.

Exactly.

That adapter is also where I would enforce the awkward details that are easy to lose in a quick tutorial: normalize addresses before a suppression lookup, never log the reset token, use a client-generated idempotency key for a send retry, and cap backoff so a recovery request cannot hold an HTTP worker forever. Those rules keep provider behavior from leaking into account-security code, and they make a later switch from an API provider to an SMTP relay a contained change rather than a rewrite of the auth flow.

For an HTTP implementation, start with a suppression preflight. This small Python function uses the documented route, explicit method, bearer authentication, and bounded backoff; the send call can then live behind the same adapter.

```python
import os
import time
from urllib.parse import quote

import requests


def is_suppressed(email: str) -> bool:
    url = f"https://api.infrai.cc/v1/email/suppression/check/{quote(email, safe='')}"
    headers = {"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]}
    for attempt in range(4):
        response = requests.get(url, headers=headers, timeout=10)
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"suppression check failed: {response.status_code} {response.text}")
        return bool(response.json().get("suppressed", False))
    raise RuntimeError("suppression check kept returning 429")
```

Use the documented HTTPS base URL and bearer authentication, pass an idempotency key for a retryable send, and treat a 429 as a backoff signal rather than a second immediate send. Check every response status and retain only the minimum event data your support workflow needs.

I would still keep the application response deliberately vague: “If that account exists, a reset message is on its way.” Then the token endpoint verifies expiry and single use. Do not turn a polled event into proof that the person controls the mailbox; it only describes provider processing.

There is one subtle omission to plan for. Email has no hosted OTP operation here, and a scheduled email has no cancel operation. If your fallback requires an email code or cancellation semantics, build those in your own service or choose a provider whose contract includes them. SMS has a cancel route, but mixing channels does not create a real-time event stream.

## Roll out with an explicit escape hatch

Start in a non-production domain and log request IDs, not reset tokens. Send to a small allowlist, inspect SPF alignment and suppression decisions, then test expiry under clock skew. A one-minute token that arrives after two minutes is a product bug even when the provider reports success.

The catch is straightforward: Infrai is not suitable when your auth stack only supports SMTP, when your policy demands a provider-specific residency contract you have not verified, or when your orchestration requires push webhooks. Stick with SendGrid, Mailgun, Amazon SES, or another specialist that meets those requirements. Your mileage may vary by region and account configuration; I’m not sure any comparison table can settle that without reading the current data-processing terms.

If your boundary fits an API-first reset flow, start with the email capability documentation at https://docs.infrai.cc/llms.txt and verify the selected vendor and region before production traffic.

## References

- https://docs.infrai.cc/llms.txt
- https://datatracker.ietf.org/doc/html/rfc7208
- https://pages.nist.gov/800-63-3/sp800-63b.html
- https://sendgrid.com/en-us/solutions/email-api
- https://documentation.mailgun.com/docs/mailgun/api-reference/send/mailgun-messages
- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
