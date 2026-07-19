# Authentication Testing Checklist

Covers login flows, password reset, multi-factor authentication, session handling, and account recovery. Pair with `checklists/session-management.md` for token lifecycle-specific tests.

## Table of Contents

- [Manual Tests](#manual-tests)
- [Automated Tests](#automated-tests)
- [Common Misconfigurations](#common-misconfigurations)
- [Burp Suite Tips](#burp-suite-tips)
- [Edge Cases](#edge-cases)
- [Common Bypasses](#common-bypasses)
- [References](#references)

## Manual Tests

### Login Flow
- [ ] Test for username/email enumeration via differing error messages ("user not found" vs. "incorrect password")
- [ ] Test for username/email enumeration via response timing differences
- [ ] Test for username/email enumeration via HTTP status code or response length differences
- [ ] Confirm account lockout exists after repeated failed attempts, and note the threshold
- [ ] Confirm lockout is per-account, not solely per-IP (per-IP-only lockout is trivially bypassed with rotating source IPs)
- [ ] Test whether the lockout mechanism itself can be abused to lock out a victim account (denial of service via forced lockout)

### Password Policy
- [ ] Confirm minimum length and complexity requirements are enforced server-side, not just client-side
- [ ] Test whether extremely long passwords are truncated silently (indicates a hashing bug, e.g., bcrypt's 72-byte limit mishandled)
- [ ] Confirm the password change flow requires the current password
- [ ] Confirm password reuse is prevented if the application claims to enforce history restrictions

### Multi-Factor Authentication (MFA)
- [ ] Confirm MFA cannot be bypassed by directly navigating to a post-login page after only completing step one (password) of a two-step flow
- [ ] Test whether the MFA code is rate-limited against brute-force
- [ ] Test whether the MFA code remains valid after successful use (should be single-use)
- [ ] Confirm the MFA enrollment flow itself requires re-authentication (prevents an attacker with a stolen session from adding their own MFA device)
- [ ] Test "remember this device" functionality for predictable or reusable tokens

### Password Reset
- [ ] Confirm reset tokens are single-use
- [ ] Confirm reset tokens expire within a reasonable window (verify the actual enforced window, not just documented policy)
- [ ] Confirm reset tokens are sufficiently random (not sequential, not derived from predictable user data like timestamp + user ID)
- [ ] Test whether the reset confirmation page/API leaks the token's validity before submission (allowing token brute-force validation separate from actual use)
- [ ] Confirm the reset flow doesn't allow host header injection to control the reset link's domain (see `payloads/host-header-injection.md`)
- [ ] Confirm successful password reset invalidates all existing sessions for the account

### Account Recovery
- [ ] Test security question-based recovery for guessable answers
- [ ] Confirm recovery via support/manual process requires adequate identity verification
- [ ] Test whether account recovery can be triggered without any proof of original account ownership (email/phone access)

## Automated Tests

### Credential Stuffing / Brute-Force Resistance
```bash
# Confirm rate limiting is actually enforced — only against your own test account, within program-permitted limits
ffuf -u https://target.example/login \
     -X POST -d 'username=testuser&password=FUZZ' \
     -w common-passwords.txt \
     -H 'Content-Type: application/x-www-form-urlencoded' \
     -fc 401
```

### Session Fixation Check
- [ ] Automate a comparison of the session identifier issued pre-login vs. post-login; they must differ

### JWT-Based Auth (if applicable)
- [ ] See `checklists/jwt.md` for the full JWT-specific checklist; at minimum confirm algorithm confusion and signature verification are not bypassable

## Common Misconfigurations

| Misconfiguration | Why It Happens | How to Spot It |
|---|---|---|
| Verbose login error messages | Developer convenience during debugging never removed | Different error text/status for valid vs. invalid usernames |
| Client-side-only password complexity enforcement | Complexity check implemented in JS for UX, never duplicated server-side | Submit a weak password directly via API, bypassing the UI |
| Session not invalidated on logout | Logout only clears client-side storage, server-side session remains valid | Capture session token before logout, replay it after logout |
| Reset token in URL with no expiry check | Token generated once, expiry field present in schema but never checked at use time | Reset token still works days after generation |
| MFA bypass via direct URL navigation | Post-login route doesn't re-verify MFA completion state server-side | Navigate directly to `/dashboard` after password step, before MFA step |

## Burp Suite Tips

- Use **Intruder** with a "Cluster bomb" or "Sniper" attack type to test lockout thresholds precisely — but throttle request rate to stay within program-permitted limits.
- Use **Comparer** to diff login error responses byte-for-byte when hunting for subtle enumeration signals (whitespace differences, header order).
- Use **Sequencer** to analyze the randomness quality of session tokens and password reset tokens.
- Set up a **Match/Replace rule** to strip or manipulate MFA-related cookies/headers to test direct navigation bypass efficiently across many requests.

## Edge Cases

- [ ] Test authentication behavior when the `Content-Type` header is manipulated (e.g., submitting JSON to an endpoint expecting form-encoded data — some frameworks parse both, creating inconsistent validation paths)
- [ ] Test behavior with Unicode username normalization (e.g., does `admin` and `аdmin` — using a Cyrillic "а" — resolve to different or the same account?)
- [ ] Test case-sensitivity consistency between registration and login for email/username fields
- [ ] Test whether OAuth/SSO login and traditional password login can be linked to the same account inconsistently, creating an authentication bypass via the weaker method

## Common Bypasses

- **Enumeration via registration flow**: even if login errors are generic, the registration flow ("email already in use") often independently confirms account existence.
- **Rate limit bypass via header manipulation**: some rate limiters key off `X-Forwarded-For`; test whether adding or rotating this header resets the counter (only within program-permitted testing volume).
- **MFA bypass via response manipulation**: for client-heavy SPAs, test whether the MFA "success" determination happens client-side based on an API response the attacker can forge or replay.
- **Password reset host header bypass**: if the reset email link is built from the `Host` header rather than a fixed configuration value, an attacker can poison the link to point to an attacker-controlled domain (see `payloads/host-header-injection.md`).

## References

- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- OWASP Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- OWASP Top 10 2021 — A07:2021 Identification and Authentication Failures: https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/
- NIST SP 800-63B — Digital Identity Guidelines: https://pages.nist.gov/800-63-3/sp800-63b.html
