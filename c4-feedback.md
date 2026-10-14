REVIEW FEEDBACK: initiate (recovery_service.go, recovery_throttle.go)

MUST FIX
1. handleResend can panic: after recoveryCache.Get fails, only ErrNotFound and ErrStoreUnavailable return; any other error falls through with a nil flow. Return 503 for any other error. Same fail-open bug in handleNewInitiate: any CheckThrottle error -> 503, not only ErrStoreUnavailable.
2. Add ErrIdentityAmbiguous to the handleNewInitiate decision table: decoy, INFO, "ambiguous". If GetIDFromCredentialsIdentifier can't return it, tell me.
3. Send failures are audited as success (real_send / resend_real stay set when GetSender or SendOtp fails). Use separate outcomes (real_send_failed, resend_send_failed), log "recovery_otp_send_failed", keep OTPRefID = "".
4. Every resolveRecipients error becomes the "no_recipients" decoy. Only ErrNoRecipients is the WARN "no_recipients" decoy. Any other error is a decoy with ERROR "recovery_lookup_failed".
5. PII in logs. Never log identifier, identityID, flow ID, OTP ref or raw error text.
   - Remove MaskCustomerId(identifier) and the identityID logs.
   - Remove the logs that print req.FlowID ("unknown flow", "max resends exceeded").
   - Log only fingerprint, category and error class (not %v of the error).
6. Confirm eaerror.ErrInternalServer maps to HTTP 503, not 500, for every Redis-down path (throttle, cache Create/Get/Update).
7. handleResend: if Update returns ErrNotFound, return 202 with a random flow_id (flow expired mid-request). ErrStoreUnavailable -> 503.
8. resolveRecipients:
   a. Add a default case to the OTPChannelType switch: config error (ERROR), not a silent decoy.
   b. EmailAndSms: filter out recipients with empty values.
   c. Config error if Notification is nil or RecoveryTemplateId is empty.
   d. Reuse the shared C2 ResolveRecipients helper if it exists.
9. recovery_throttle.go:
   a. `limit` is only used in a debug log. Remove it, or make CheckThrottle return (allowed bool, count int64, err error) so one place owns the decision.
   b. It logs the full key (client IP and identifier hash). Log only the key type plus a short fingerprint.
   c. Fix the comment: it is a fixed window, not a sliding window.

SHOULD FIX
10. Extract one helper for identity check -> resolve recipients -> send OTP, returning (isDecoy, outcome, otpRefID), used by both paths.
11. handleNewInitiate calls GetIdentity twice. Reuse the first result.
12. Store decoy flows with IdentityID = "".
13. Use one `now` for LastSentAt/CreatedAt; use ctx.Request.Context() consistently.
14. Use one constant for the response message ("If the account exists, a verification code has been sent.") instead of separate RecoveryOTPMessage and ResendOTPMessage.
15. handleResend: a GetIdentity error is ERROR "recovery_lookup_failed". Inactive or azure_oidc is INFO "disabled_mid_flow".
16. The cooldown 429 only puts the seconds in the message text. Add retry_after (seconds) to the body.
17. Add an audit outcome on the 503 paths (e.g. "store_unavailable").

CONFIRM (tell me where)
18. Lines 394-407 of handleResend (collapsed): they must set Decoy = true and OTPRefID = "", log at the right level, and not log identityID.
19. Where is the per-IP throttle with the count > MaxInitiatePerIP15m check? It must run before the identity lookup.
20. Where is the minimum response time applied? addRandomJitter alone is not a floor. It must cover every 202 path in both flows.
21. The identifier hash must use the same trimmed identifier in initiate and in handleResend.

TESTS TO ADD
- Ambiguous lookup: 202, decoy, INFO, SendOtp not called.
- Send failure (new and resend): 202, OTPRefID "", Decoy=false, outcome send_failed, ERROR logged.
- resolveRecipients: ErrNoRecipients vs config error give different category and level; unknown channel; empty EmailAndSms value.
- Resend: Get returns an unexpected error -> 503, no panic. Update returns ErrNotFound -> 202 with a random ID.
- Resend: cooldown -> 429 with retry_after; max resends -> 429; decoy and real flows behave the same.
- Log-capture test: identifier, identityID, flow ID and OTP ref never appear in any log line, across all paths.
- 4th initiate for one identifier -> 429; any throttle error -> 503.
