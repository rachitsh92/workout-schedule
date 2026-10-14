REVIEW FEEDBACK: ENTAUTH-2188 ValidateRecovery (recovery_service.go)

MUST FIX
1. Confirm eaerror.ErrInternalServer maps to HTTP 503. Every Redis, lockout, sender and Kratos failure path uses it. If it maps to 500, use the constructor that returns 503.
2. Raw error text in logs. These log %v of an error: cacheErr, lockoutErr, recordErr, delErr, senderErr, verifyErr, clearErr, tokenErr. Log only the error class (e.g. "cache_unavailable", "lockout_error", "sender_error").
3. The unknown-tenant and recovery-disabled logs print req.TenantID, which is attacker-controlled input. Log a fixed category only.
4. Concurrent double-validate. Two simultaneous requests with the same flow_id and the correct OTP can both pass VerifyOtp, and each then creates a reset token.
   - First check whether sender.VerifyOtp invalidates the OTP on success.
   - If it does not, add an atomic guard so only one request reaches the success path (for example a SetNX in-flight key per flow). Alternatively propose a small atomic ConsumeFlow on the cache, but do not change RecoveryCache without my approval.
   - Add a test with 10 concurrent correct-OTP requests: exactly one gets 200.
5. Line 797: a GetIdentity error is audited as outcomeSenderError. Use a lookup category (e.g. outcomeLookupFailed).
6. Line 752: auditLogger.Add(OUTCOME, "") before the real outcome. Remove it and set the outcome once per exit (e.g. a deferred setter), so there is no blank or duplicate outcome.

SHOULD FIX
7. The RecordFailure -> locked? -> delete flow -> 429, else 401 block is repeated three times (decoy/empty OTPRefID, malformed OTP, wrong OTP). Extract one helper so they cannot diverge.
8. Input validation only checks for empty values. Add length caps on tenant_id, flow_id and otp (e.g. 256) -> 400.
9. Delete errors on the 429 paths are only logged, which is fine. Add a comment saying that this is intentional.

TESTS TO ADD
- 10 concurrent correct-OTP requests: exactly one 200.
- IsLocked, RecordFailure and ClearAttempts all receive "recovery:" + flow.IdentifierHash.
- Decoy flow and real flow follow the same lockout progression (same Nth failure -> 429).
- A log-capture test: the OTP, flow_id, reset token, identifier, identityID and OTPRefID never appear in any log line, including the error paths.
- Redis, lockout, sender and Kratos failures return 503 and do not call RecordFailure.


---------------------------------------------------------------------

FIX: ENTAUTH-2188 in-flight guard released too early (recovery_service.go)

PROBLEM
In attemptRecoveryValidationWithSender, `defer DeleteInFlight` fires when that function returns. completeRecoveryServiceSuccess (ClearAttempts, flow Delete, CreateResetToken) runs after it. So the guard is gone while the flow still exists, and a second request can acquire it, load the same flow and call VerifyOtp again. That burns a lockout attempt for the real user, or issues a second reset token if the OTP isn't single-use at the sender.

FIX
1. Move the guard's lifetime up to ValidateRecovery. Acquire it (TryConsumeFlow) right before sender.VerifyOtp, and release it with ONE defer in ValidateRecovery, so it is released only after completeRecoverySuccess has returned (flow deleted, token created) or after a failure path returns.
2. Keep the structure: claim -> VerifyOtp -> handle failure -> completeRecoverySuccess, all inside the scope of that one defer. Do not release between VerifyOtp and completeRecoverySuccess.
3. If the claim is not acquired, keep returning the generic 401 and do NOT call RecordFailure.
4. Do not change RecoveryCache signatures.

QUICK CHECKS (change only if they fail)
- The in-flight key has a short TTL (e.g. 30s) so a crash cannot block a flow for long.
- eaerror.ErrServiceUnavailable returns HTTP 503. Add one test asserting the status.

TESTS
- 10 concurrent correct-OTP requests with the same flow_id: exactly one 200, one reset token created, no RecordFailure for the losers.
- A request arriving after VerifyOtp succeeded but before the flow is deleted does not reach VerifyOtp.
- After a failed attempt (wrong OTP), the guard is released and a retry with the correct OTP works.
- After a success, a second validate returns 401 (flow gone).
