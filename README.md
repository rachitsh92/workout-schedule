# workout-schedule
TASK: Implement the Recovery Cache and Configuration Layer (storage only, NO API endpoints)
Repo: enterprise-authentication-service (Go). This is the base for 3 later recovery endpoints (initiate, validate, reset).

STEP 0 - DISCOVER BEFORE CODING (read, don't change)
- cache/password_lockout_cache.go: copy its conventions (struct layout, constructor, key helpers, logging, go:generate mockgen line, test style).
- Find the interface that declares ExecuteLuaScript (used as c.redisStore.ExecuteLuaScript(ctx, script, []string keys, []interface{} args)). Note its exact return type and which interface/type it lives on. Use whatever the lockout cache uses.
- Check how redis_util.RedisStore.Get reports a missing key (redis.Nil? custom error?) and whether Set/Get JSON-marshal values or need raw strings.
- DO NOT modify the shared RedisStore interface.
- Report what you found before writing code.

FILES
1. configuration/kratos_config.go
   - Add RecoveryPolicy struct with JSON tags: enabled (bool), flow_ttl_seconds, reset_token_ttl_seconds, resend_cooldown_seconds, max_resends, max_initiate_per_identifier_15m, max_initiate_per_ip_15m (all int).
   - Add the RecoveryPolicy field to KratosConfig (json:"recovery,omitempty"; pointer or value, whichever fits the existing config style).
   - Add func (k *KratosConfig) GetRecoveryPolicy() RecoveryPolicy: a value of 0 means unset, so apply defaults: flow TTL 600, reset token TTL 300, resend cooldown 30, max resends 3, max initiate per identifier 3, per IP 20. Enabled has NO default (false). A missing recovery block returns Enabled=false. Never mutate the stored config; work on a copy.

2. models/recovery.go
   type RecoveryFlow struct {
     TenantID, IdentityID string; IdentifierHash string (SHA256 of tenant+":"+trimmed identifier);
     Decoy bool; OTPRefID string; Resends int; LastSentAt, CreatedAt time.Time }
   with json tags tenant_id, identity_id, identifier_hash, decoy, otp_ref_id, resends, last_sent_at, created_at.

3. cache/recovery_cache.go
   - Define interface RecoveryCache + //go:generate mockgen directive like the lockout cache, and a Redis-backed implementation using redis_util.RedisStore plus the Lua executor found in Step 0. Constructor in the same style as the lockout cache.
   - Methods:
     Create(ctx, flow *RecoveryFlow, ttl time.Duration) (flowID string, err error)
     Get(ctx, flowID string) (*RecoveryFlow, error)
     Update(ctx, flowID string, flow *RecoveryFlow) error
     Delete(ctx, flowID string) error
     CreateResetToken(ctx, tenantID, identityID string, ttl time.Duration) (token string, err error)
     PeekResetToken(ctx, token string) (tenantID, identityID string, err error)
     ConsumeResetToken(ctx, token string) (tenantID, identityID string, err error)
   - Keys: recovery:flow:<flowID> (value = flow JSON); recovery:reset:<token> (value = "tenantID:identityID"). No PII in keys.
   - IDs: flowID and token = 32 bytes from crypto/rand, base64.RawURLEncoding. NEVER math/rand. If rand.Read fails, return ErrStoreUnavailable.
   - Create/CreateResetToken: store with TTL using SetNX (a collision must never overwrite). On collision, retry with a new ID (max 3 tries).
   - Update: MUST NOT extend TTL and MUST NOT resurrect an expired key. Use a Lua script: PTTL key; if ttl <= 0 return 0; else SET key ARGV[1] PX ttl; return 1. Return ErrNotFound when the script returns 0.
   - ConsumeResetToken: MUST be atomic single-use. Use a Lua script: v = GET key; if v then DEL key end; return v. Do NOT implement as Get then Delete. Missing key / nil result -> ErrNotFound.
   - Peek: plain Get, no delete.
   - Parse "tenantID:identityID" with SplitN(s, ":", 2); malformed -> ErrNotFound.
   - Errors: exported sentinels ErrNotFound (expired, missing, already used) and ErrStoreUnavailable (any Redis/transport error, wrapped with %w so errors.Is works). Callers will map ErrNotFound -> 401 and ErrStoreUnavailable -> 503, so the two must never be confused. Treat Redis errors as fail-closed.
   - Do not log tokens, flow IDs, identifiers or flow contents.

TESTS (match the lockout cache test style; use the same Redis test approach, e.g. miniredis if the repo uses it, otherwise a mock/embedded setup)
- Config: zero-value policy gets all defaults; explicit values are kept; no recovery block -> Enabled=false; stored config not mutated.
- Create then Get round-trips all fields; Get of unknown ID -> ErrNotFound.
- Update changes the data and the remaining TTL is not extended (check TTL before/after; allow small tolerance); Update on a missing/expired flow -> ErrNotFound and does not create the key.
- Delete removes the flow.
- CreateResetToken returns a 43-char URL-safe string; 1000 tokens are all unique.
- Peek twice succeeds; Consume then Consume again -> second returns ErrNotFound.
- AC1: ConsumeResetToken under 10 concurrent goroutines on one token succeeds exactly once, the other 9 get ErrNotFound.
- Redis failure (closed/erroring store) -> ErrStoreUnavailable for every method, never ErrNotFound.
- Expired token -> ErrNotFound.

ACCEPTANCE CRITERIA
1. RecoveryPolicy in config with the defaults helper.
2. RecoveryFlow model in models/recovery.go.
3. RecoveryCache implemented in cache/recovery_cache.go, with a generated mock.
4. ConsumeResetToken is single use, atomic, proven by the concurrency test.
5. Update never extends TTL.
6. IDs/tokens come from crypto/rand, 32 bytes, unique.
7. go build, go vet and all tests pass; existing tests unaffected.

OUT OF SCOPE: HTTP handlers, routes, throttling/rate limiter, notification sending, Kratos calls.
