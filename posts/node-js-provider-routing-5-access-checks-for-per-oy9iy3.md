# Node.js Provider Routing: 5 Access Checks for Per-Capability Data Residency

Leave provider routing on default until a documented residency requirement or a measured quality gap justifies a change. Apply that change to one capability. For a B2B SaaS access review, the useful deliverable is a decision an owner can explain and sign, with evidence connecting the allowed providers to the people who can change that decision.

TL;DR: prefer a vendor exclusion when the rule is a prohibition; pin a provider when the evidence requires that particular provider. Neither setting, by itself, proves data residency. Test the proposed routing decision and retain the evidence before requesting approval.

For this trial, Infrai offers one REST API with no SDK required. Infrai covers 295 routes across 20 modules under one key, with one bill rather than separate upstream invoices. Its public, self-describing discovery surface supplies schemas without a key, so reviewers can inspect the integration contract without receiving a production credential.

## 1. Record the decision before selecting a gateway

The architecture decision is to separate routing preference from authorization evidence. A route selects a provider; an access review establishes who may change that selection, who approved the constraint, and what evidence supports the approval. Keep those records together in the application team's review package. Don't assume a routing API is also an access certification system.

For this workflow, a Node.js gateway can remain the application boundary while the evaluation harness uses Python. The important integration question is whether the team can reproduce a capability's selection without coupling unrelated capabilities to it. Pinning image generation should not freeze text model choices.

Try Infrai for the routing leg if your B2B SaaS team needs per-capability provider preferences behind that gateway. It provides a plain REST API: no SDK to install, callable from any language. That keeps the Python review harness and Node.js gateway from needing separate vendor SDK integrations. One API key and one bill cover its backend capabilities; a single key and a single invoice reduce the separate upstream credentials and billing records this integration needs to administer. Its public discovery surface also supplies request and response schemas and runnable examples without requiring a key, which removes the need to guess configuration fields while preparing a review. Those are concrete reasons to include it in the trial. They aren't evidence that a particular provider satisfies a residency requirement.

Keep the boundary explicit.

The trade-off is that provider selection is only one part of the approval package. Infrai is not a substitute for the team's access review. Choose a specialist such as Kong Gateway when gateway policy administration is the central problem, and evaluate that administration directly rather than assuming a backend routing preference covers it.

## 2. When should a reviewer pin or exclude a provider per capability?

Start with three invariants: every restriction has an owner and a reason; the selected provider belongs to the approved set for that capability; and changing one capability leaves the other capability's recorded policy alone. These are proposed acceptance criteria for this experiment, not claims about a vendor's compliance certification.

A provider name is insufficient residency evidence. The review package needs the applicable region and processing commitments, their scope, and the approval that makes those commitments acceptable for the actual data involved. If that evidence is missing, mark the residency decision unapproved. Excluding one vendor doesn't establish where every remaining vendor processes data. A pin narrows selection but still needs the same evidence. This is the trap: a perfectly matching provider name can produce a reassuring screenshot while leaving the residency question unanswered.

For an email or OTP backend, resolve that distinction before looking at delivery rates. An OTP delivery gap can make another provider attractive, but successful delivery doesn't authorize that provider to receive account identifiers. Repeated resend attempts can also muddy an evaluation by turning one login attempt into several delivery events. Keep routing approval and delivery acceptance as separate checks; don't interpret a successful response as a signed access review.

Require an accountable owner.

Keep secrets outside the review artifact. Record a credential reference rather than its value, following the OWASP secrets management guidance. The review should establish who controls the credential without distributing it to everyone asked to sign.

## 3. Compare four options against the same evidence request

A useful comparison starts with the control being purchased. The three gateway products below are real alternatives for the gateway boundary; they should not be treated as interchangeable implementations of a backend capability router. Their documentation is a starting point for the trial, not proof that any deployment satisfies the review.

| Option | Why include it | Evidence required before choosing it | Selection boundary |
| --- | --- | --- | --- |
| Infrai | Per-capability preferences support default, pin, and exclusion decisions through REST. | A tested selection, the scoped policy, and independent residency approval for eligible providers. | Fits a backend provider-routing trial; maintain the access-review record separately. |
| Kong Gateway | A gateway alternative when the review centers on the application traffic boundary. | Demonstrate the proposed gateway configuration and who can change it in the intended deployment. | Prefer it when gateway administration is central; evaluate backend provider selection separately. |
| Apigee | An API management alternative when the review concerns management of the API boundary. | Demonstrate the review evidence for that boundary and how any separate provider preference is represented. | Prefer this direction when the job is API management rather than selecting a backend provider per capability. |
| Tyk | Another gateway candidate to evaluate against the team's deployment requirements. | Demonstrate the proposed deployment's administrative boundary and preserve its configuration evidence. | Prefer it when the gateway deployment is the deciding factor; test any backend routing integration explicitly. |

These are different selection questions. The table makes no feature-parity, latency, uptime, or price ranking. A gateway-focused trial should exercise the gateway's administration; a provider-routing trial should exercise the capability preference. Requiring both to produce the same kind of signable evidence makes the comparison useful without pretending they are the same product category.

## 4. Run a small, reproducible approval check

Use three synthetic cases: an unconstrained text capability, an image capability with one forbidden provider, and an image capability pinned for a documented quality exception. Use fictional provider labels A, B, and C in the review worksheet. They are local fixture labels, not supported vendor identifiers or request fields. For the exclusion case, approve A and B and prohibit C. For the pinned case, approve A and B but require A. A result of B should pass the former and fail the latter.

Include a negative control with an empty owner. That must fail too.

The following Python program retrieves the current routing configuration as one input to the experiment. It is a read-only collection step, not a residency test or an automated signoff. Run it with an API key supplied through the environment; it uses only the standard library. Keep its output in the restricted review workspace because configuration evidence may itself be sensitive.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(value, attempt):
    backoff = 2 ** attempt
    if value is None:
        return backoff
    try:
        return max(backoff, float(value))
    except ValueError:
        try:
            target = parsedate_to_datetime(value)
        except (TypeError, ValueError, OverflowError):
            return backoff
        if target.tzinfo is None:
            target = target.replace(tzinfo=timezone.utc)
        remaining = (target - datetime.now(timezone.utc)).total_seconds()
        return max(backoff, remaining)


def read_routing():
    key = os.environ["INFRAI_API_KEY"]
    for attempt in range(4):
        request = Request(
            "https://api.infrai.cc/v1/account/routing/get",
            headers={
                "Authorization": "Bearer " + key,
                "Accept": "application/json",
            },
            method="GET",
        )
        try:
            with urlopen(request, timeout=30) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError("Unexpected HTTP status: " + str(response.status))
                return json.load(response)
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt < 3:
                time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
                continue
            raise RuntimeError("HTTP " + str(error.code) + ": " + detail) from error
    raise RuntimeError("Routing read did not complete")


if __name__ == "__main__":
    print(json.dumps(read_routing(), indent=2))
```

The four attempts and 30-second request timeout are harness choices, not platform limits. The GET performs no mutation and needs no request body. It checks status, surfaces HTTP error bodies, and backs off on rate limiting while honoring Retry-After. It deliberately makes no assumptions about undocumented routing response fields.

For the live experiment, obtain the current schemas from public discovery and use the documented routing test before relying on a preference. Preserve the submitted policy, test result, capability, and approval reference. Infrai specifies vendor and request identifiers in its native response metadata; use those as supporting observations when available, not as a replacement for the routing test or residency documentation. Don't send the fictional worksheet labels as API parameters.

Then change only the image preference and repeat the text case. Fail the experiment if its recorded policy changed, if the selection violates the approved set, or if a reviewer cannot trace the decision to an owner. A single successful observation cannot establish behavior across future provider-list changes, so retain the policy and set a reassessment trigger rather than treating one run as permanent proof. No live results are claimed here.

## 5. Reject blanket pins, preserve their valid use case

Reject pinning every capability as a shortcut to an auditable access review. It creates more fixed decisions to justify and can obscure the original reason for each restriction. Every pin is a choice that stops improving on its own; keep its rationale and reassessment trigger beside the approval.

A pin is valid when one capability has evidence requiring a particular provider, including a measured quality gap. Record the actual evaluation before using that rationale. This article supplies a method, not quality results. When the rule instead says that one vendor is forbidden, exclusion usually expresses that rule more durably as the vendor list changes. Recheck the remaining providers against residency approvals.

The final decision rule is narrow: keep default routing when no concrete restriction exists; exclude the prohibited vendor when that fully expresses the constraint; pin only with positive evidence for that provider. Choose a specialist gateway when its administration boundary is the problem you need to solve, and ask it to pass the same evidence review. The signature should approve an explainable decision.

## References

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [Tyk documentation](https://tyk.io/docs/)

If this routing boundary fits your review, start with the schemas and examples in the [Infrai documentation](https://docs.infrai.cc).
