# Zero Trust Conditional Access Baseline in Microsoft Entra ID

**Platform:** Microsoft Entra ID (P2) · Conditional Access · Identity Protection
**Domain:** Identity & Access Management · Zero Trust enforcement
**Approach:** Safe rollout — break-glass account, report-only validation, What If testing before enforcement

---

## Summary

Zero Trust means *never trust, always verify* — every access request is verified (identity, device, location, risk) as if it came from an untrusted network, regardless of where it originates. Conditional Access is how Microsoft Entra **enforces** that: each sign-in is evaluated against a set of Condition → Control policies, and access is granted, blocked, or challenged based on the signals.

I designed and deployed a **six-policy Zero Trust baseline**, built it the professional way — behind a break-glass account, in report-only mode — and validated every policy with the What If simulator before enforcing. I then confirmed live enforcement by watching a policy force MFA registration on a real sign-in.

**Result:** a complete, tested Conditional Access baseline mapping to the three Zero Trust pillars — verify explicitly, least privilege, assume breach — with each policy proven to fire only on its intended condition.

---

## Zero Trust, and how Conditional Access enforces it

The old "castle-and-moat" model trusted anyone inside the network perimeter. Zero Trust discards that — there is no trusted inside. Microsoft's model rests on three pillars:

- **Verify explicitly** — authenticate and authorise on all signals: identity, device, location, risk.
- **Least privilege** — the minimum access needed, with the tightest controls on the highest-value accounts.
- **Assume breach** — act as if an attacker is already in; close the paths they would use.

Every Conditional Access policy is a single sentence: **Conditions (IF) → Access Controls (THEN)**. The baseline below operationalises the three pillars.

| Policy | Condition → Control | Pillar |
|---|---|---|
| CA01 | All users → require MFA | Verify explicitly |
| CA02 | Legacy auth clients → block | Assume breach |
| CA03 | Admin roles → MFA + stricter session | Least privilege |
| CA04 | Untrusted country → block | Verify explicitly |
| CA05 | Risky sign-in → require MFA | Verify explicitly (adaptive) |
| CA06 | Risky user → force password reset | Assume breach (adaptive) |

---

## Safety first — the non-negotiables

Conditional Access can lock an administrator out of their own tenant. Two safeguards were put in place *before* any policy was written:

**A break-glass account** — one emergency Global Administrator, cloud-only and excluded from every policy, so there is always a way back in if a policy misfires. In production you keep two; sign-ins to it are monitored, because it should almost never be used.

![Break-glass admin account](https://github.com/KevoT0/Entra-Conditional-Access/blob/main/2.png)

The break-glass account sits in a dedicated exclusion group that every policy excludes — so the emergency account is protected in one place, not policy by policy.

![CA-Exclude-BreakGlass group](https://github.com/KevoT0/Entra-Conditional-Access/blob/main/1.png)

**Report-only mode** — every policy was created in report-only first. It logs what it *would* do without enforcing, so the impact can be validated before it can break anything. Report-only catches problems *before* they happen; the break-glass account is the escape hatch *if* one slips through. Belt and braces.

---

## The baseline policies

### CA01 — Require MFA for all users (verify explicitly)

The cornerstone. A password can be phished or bought; MFA demands a second, independent proof of identity on every sign-in. Targeted at all users, with the break-glass group excluded.

![CA01 policy summary](https://github.com/KevoT0/Entra-Conditional-Access/blob/main/6.png)

### CA02 — Block legacy authentication (assume breach)

Legacy protocols (POP, IMAP, SMTP AUTH, older clients) use basic authentication and **cannot perform MFA** — so an attacker with a stolen password uses them to bypass MFA entirely. The condition targets exactly those legacy client types, and the control blocks them.

![CA02 client apps condition](https://github.com/KevoT0/Entra-Conditional-Access/blob/main/7.png)

![CA02 policy summary](https://github.com/KevoT0/Entra-Conditional-Access/blob/main/8.png)

### CA03 — Stricter controls for admins (least privilege)

Admins are the highest-value target, so they get more than baseline MFA. On top of MFA, this policy adds **session controls** — a 4-hour sign-in frequency and no persistent browser session — so a stolen admin session dies quickly and can't be "remembered."

![CA03 session controls](https://github.com/KevoT0/Entra-Conditional-Access/blob/main/10.png)

![CA03 policy summary](https://github.com/KevoT0/Entra-Conditional-Access/blob/main/11.png)

Grant controls decide whether you get *in* (MFA/block); session controls decide what happens *during* the session (how often you re-verify, whether you're remembered). CA03 uses both.

### CA04 — Block access from untrusted locations (verify explicitly)

A named location defines the trusted geographies; the policy then blocks everywhere else. The logic is *include Any location, exclude Trusted Countries* — mathematically leaving only untrusted locations, which get blocked.

![Trusted Countries named location](https://github.com/KevoT0/Entra-Conditional-Access/blob/main/12.png)

![CA04 excluding trusted countries](https://github.com/KevoT0/Entra-Conditional-Access/blob/main/13.png)

### CA05 — Require MFA for risky sign-ins (adaptive)

Identity Protection scores each sign-in for risk (impossible travel, anonymous IP, unfamiliar properties). A High or Medium risk sign-in is challenged for MFA even if the password was correct — verification that *scales with risk*.

![What If — legacy auth](https://github.com/KevoT0/Entra-Conditional-Access/blob/main/17.png)

![CA05 sign-in risk condition](https://github.com/KevoT0/Entra-Conditional-Access/blob/main/18.png)

### CA06 — Force password reset for risky users (adaptive)

Where sign-in risk is about a suspicious *event*, user risk is about a likely-compromised *identity* (e.g. credentials found leaked). A High-risk user is forced through a secure password change (with MFA) to lock the attacker out.

![CA06 user risk condition](https://github.com/KevoT0/Entra-Conditional-Access/blob/main/19.png)

![What If — legacy auth](https://github.com/KevoT0/Entra-Conditional-Access/blob/main/28.png)

---

## Validation — proving every policy with What If

Before enforcing, each policy was validated with the **What If** simulator, which evaluates a hypothetical sign-in against all policies. The tests isolate one trigger at a time — everything else held constant (Amara, Windows, Browser, UK) — so each result proves *that* policy fires on *that* condition, and correctly does **not** fire otherwise.

**Untrusted location (Russia IP) → CA01 + CA04 both apply** — MFA required *and* the sign-in blocked:

![What If — Russia](https://github.com/KevoT0/Entra-Conditional-Access/blob/main/14.png)

**Trusted location (UK IP) → only CA01 applies** — CA04 correctly drops off. Same user, same app, different location, different decision. That contrast is Zero Trust adapting to signal:

![What If — UK](https://github.com/KevoT0/Entra-Conditional-Access/blob/main/15.png)

**Legacy client → CA01 + CA02 apply** — the legacy-auth block fires only for legacy client types:
![What If — legacy auth](https://github.com/KevoT0/Entra-Conditional-Access/blob/main/20.png)

![What If — legacy auth](https://github.com/KevoT0/Entra-Conditional-Access/blob/main/21.png)

**Admin user → CA03 applies with its session controls** — the admin policy fires only for admins (it did not fire for the normal users), proving role-scoping works:

![What If — legacy auth](https://github.com/KevoT0/Entra-Conditional-Access/blob/main/22.png)

![What If — admin](https://github.com/KevoT0/Entra-Conditional-Access/blob/main/23.png)

**Risky sign-in (High) → CA01 + CA05 apply** — adaptive MFA on a risky event:

![What If — legacy auth](https://github.com/KevoT0/Entra-Conditional-Access/blob/main/24.png)

![What If — risky sign-in](https://github.com/KevoT0/Entra-Conditional-Access/blob/main/25.png)

**Risky user (High) → CA01 + CA06 apply** — forced password reset on a compromised identity:

![What If — legacy auth](https://github.com/KevoT0/Entra-Conditional-Access/blob/main/26.png)

![What If — risky user](https://github.com/KevoT0/Entra-Conditional-Access/blob/main/27.png)

Six policies, six isolated tests, each firing exactly on its intended condition and staying silent otherwise. This is how Conditional Access is validated before it is enforced.

---

## Live enforcement

With validation complete, CA01 was switched from report-only to **On**, and a normal user signed in. The policy stopped the sign-in and forced MFA registration — the difference between a policy that *exists* and one that *works*:

![Live enforcement — MFA forced](https://github.com/KevoT0/Entra-Conditional-Access/blob/main/16.png)

In report-only the user would have signed in with no prompt (logged only). Enforced, the policy physically requires MFA before granting access — the report-only → enforce rollout completed.

---

## Key design decisions

- **Break-glass + report-only, always.** No Conditional Access policy was created without the exclusion group and report-only validation. This is the discipline that prevents tenant lockout.
- **Scoped, not blanket.** Each policy targets only its intended condition (legacy clients, admin roles, risk levels), proven by the What If tests showing policies fire selectively.
- **Grant vs session controls.** Admin hardening uses session controls (re-auth frequency, no persistence), not just grant controls — limiting the lifespan of a stolen session.
- **Adaptive layer (P2).** Risk-based policies (CA05, CA06) verify in proportion to real-time risk — the adaptive core of Zero Trust, enabled by Entra ID P2 / Identity Protection.
- **Identity signals feed detection.** Every sign-in these policies evaluate produces telemetry that a SOC hunts in Microsoft Sentinel — Conditional Access is the identity control plane detection sits on.

---

## Future improvements

- **Policy-as-code** — export and deploy the policies via Microsoft Graph PowerShell (`New-MgIdentityConditionalAccessPolicy`), version-controlled in Git, so the baseline is repeatable and auditable rather than click-configured.
- **Device compliance** — require Intune-compliant or hybrid-joined devices, adding the device pillar to the location and risk signals.
- **Authentication strength** — require phishing-resistant methods (FIDO2, certificate) for admins instead of any MFA.
- **Staged enforcement** — move each policy report-only → enforced on a pilot group → tenant-wide, monitoring sign-in logs at each stage.

---

## Skills demonstrated

· Zero Trust architecture and the three-pillar model
· Conditional Access policy design (Condition → Control)
· Entra ID P2 / Identity Protection (risk-based access)
· Safe rollout — break-glass, report-only, What If validation
· MFA enforcement, legacy-auth blocking, location and session controls
· Least-privilege admin hardening
· Identity as the control plane for SOC detection
