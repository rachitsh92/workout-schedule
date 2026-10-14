FIX: ENTAUTH-2189 ResetRecovery (recovery_service.go), functional blockers only

1. A failed Kratos update returns 200.
   completePasswordReset returns false when UpdateIdentityWithPassword fails, and ResetRecovery treats that as "sessions not revoked": it audits reset_success and returns 200 "password_reset" even though the password was not changed.
   FIX: return (sessionsRevoked bool, err *eaerror.EaError) from completePasswordReset. On an update failure return eaerror.ErrServiceUnavailable (503) with audit outcome update_failed, and skip the patch, lockout reset, revocation and notification.

2. Revocation failure skips the notification.
   The `return false` after the revoke error exits before sendPasswordChangedNotification.
   FIX: on a revoke failure log the ERROR "recovery_revocation_failed", set sessionsRevoked=false, and continue to the notification. The response stays 200.

3. Wrong session revoker.
   The ticket requires SessionRevoker.RevokeAll(ctx, tenant, identityID, identifiers, "password_reset") (Kratos + SMS + Hydra). The code calls s.identity.ClearAllSessions(tenantID, identityID).
   FIX: check what ClearAllSessions does. If it is not the C1 SessionRevoker covering Hydra and SMS, replace it with SessionRevoker.RevokeAll, otherwise old tokens survive a password reset.

4. The 400 has no structured violations list.
   It sends only the message "Password does not meet policy: a, b".
   FIX: return JSON {"error_message": "Password does not meet policy", "violations": ["min_length", ...]} (rule IDs only). Use or add an eaerror constructor that carries the list.

5. Verify must_change_password is actually patched.
   patchMustChangePassword reads traits["metadata_admin"], but metadata_admin is normally a top-level Kratos field, not a trait. If GetIdentity doesn't return it there, the patch never runs and the user is forced to change the password again at next login.
   FIX: add a test against the real GetIdentity output, and if metadata_admin isn't exposed, fetch it or patch without that check.

TESTS
- UpdateIdentityWithPassword fails: 503, no lockout reset, no revocation, no notification.
- RevokeAll fails: 200 with sessions_revoked=false and the notification is still sent.
- 400 response contains the violations array and the token is still valid on retry.
