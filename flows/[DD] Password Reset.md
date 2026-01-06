# MSP Password Reset Automation: Findings & Strategy

## 1. Problem Statement: The "Identity Trust" Gap

Despite password resets accounting for **~52% of L1 ticket volume**, 70-80% remain manual. The bottleneck is not the *reset* (which APIs do easily) but the **identity verification**.

Current automation fails because:

- **The Verification Paradox:** You cannot verify a user via MFA if the reason they are calling is that they *lost* their MFA device.
- **The Data Gap:** Many MSPs lack "out-of-band" contact data (mobile numbers) in Azure AD, making automated verification impossible.
- **The "Human Firewall" Failure:** L1 engineers often bypass security protocols (e.g., trusting a voice) to provide "white glove" service, creating social engineering vulnerabilities (e.g., Scattered Spider attacks).

## 2. The "Standardized MSP" Prerequisite

**Critical Strategy:** As an MSP Rollup, we enforce a strict operating standard upon acquisition. We do not design automation for "messy data"; we fix the data first.

**Mandatory Enforcement:**

1. **Mobile Numbers in Azure AD:** Every user must have a verified mobile number populated in the `mobilePhone` field.
2. **MFA Registration:** Every user must be registered for Microsoft Authenticator (or equivalent) during onboarding.

By enforcing these two rules, we eliminate "Ghost Users" and insecure "SMS-only" paths, simplifying our automation logic to just **two robust scenarios**.

## 3. The 2-Path Verification Logic

With data hygiene enforced, the complex 4-path logic collapses into a simple binary decision: **"Does the user have access to their trusted device?"**

| Path | Scenario | Method | Automation Rate |
| --- | --- | --- | --- |
| **Path A** | **Has Device** (Forgot Password) | **MFA Push:** Trigger push notification to user's Authenticator App. **Cryptographically Secure.** | ~80% |
| **Path B** | **Lost Device** (MFA Lockout) | **Manager Approval (Hybrid):** <br>1. Send **Teams Adaptive Card** to manager.<br>2. Fallback to **SMS** if offline/no reply in 5 min. | ~20% |
| **Escalation** | **All Failed** | **L1 Engineer:** Assign ticket for manual identity proofing. | <1% |

### Why This is Superior

1. **Eliminates Phishable SMS Codes:** We no longer send 6-digit codes (which can be intercepted/phished). We rely only on App Pushes (Path A) or Manager Trust (Path B).
2. **Reduces Engineering Debt:** We build and maintain only 2 workflows instead of 4.
3. **Forcing Function:** The requirement to "clean up AD data" improves the overall security maturity of the acquired MSP.

## 4. Why Automating Manager Approval is Safer Than Voice Calls

Replacing human phone calls with Teams/SMS automation is not just faster—it is **more secure** for three critical reasons:

1. **Anti-Social Engineering:** Attackers rely on "social engineering"—charming, bullying, or pressuring L1 engineers to bypass protocols (e.g., "I'm the CEO, just do it!"). An automated workflow cannot be charmed or bullied; it strictly enforces the verification step every time.
2. **Audit Trail:** A phone call leaves no digital footprint other than a subjective ticket note ("Called Bob, he said OK"). An automated flow logs the exact time, the specific device/number messaged, the IP address of the approver, and the precise approval token used. This creates a defensible compliance audit trail.
3. **Verification of the Verifier:** When an L1 engineer calls a manager, they often implicitly trust the voice or the number they dialed manually. An automated system pulls the *exact* registered contact method (Teams ID or Mobile) from the Source of Truth (Azure AD), ensuring the approval request goes to the real manager, not an impersonator.
4. **Consistency Under Pressure:** L1 engineers may feel pressured to skip the manager call if the user is angry or claims urgency ("I have a board meeting in 5 minutes!"). An automated system never skips the verification step, regardless of the user's emotional state.

## 5. Implementation

- **Step 1:**  Try MFA push for user identity verification
- **Step 2:** Try manager approval for user identity verification

![password-reset-flow.png](res/password-reset-flow.png)

---

---

## 6. Expected Outcomes?

| Metric | Before Automation | After 2-Path Logic |
| --- | --- | --- |
| **Escalation Rate** | ~30% | <1% |
| **L1 Effort per Reset** | 10 minutes | 0 minutes (fully automated) |
| **Automation Success Rate** | ~20% | ~99% |
| **Security Posture** | Vulnerable to social engineering | Consistent, auditable verification |