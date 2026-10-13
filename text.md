TASK: Fix PeekResetToken in cache/recovery_cache.go (bug found in review)

PROBLEM
CreateResetToken stores the raw string "tenantID:identityID" via Lua (SET key value PX ttl NX).
PeekResetToken reads it with c.redisStore.Get(ctx, key, &value), which JSON-unmarshals the stored value.
A raw string is not valid JSON, so against a real Redis Peek fails with:
  "recovery: store unavailable: invalid character 'c' looking for beginning of value"
Every reset endpoint call (ENTAUTH-2189 peeks first) would return 503.
The unit test misses this because it mocks redisStore.Get with SetArg, so the JSON step never runs.
ConsumeResetToken is fine: it reads the raw value through Lua.

FIX (do exactly this)
1. In cache/recovery_cache.go, add:
   const peekResetTokenScript = `return redis.call("GET", KEYS[1])`

2. Rewrite PeekResetToken to read the raw value the same way ConsumeResetToken does:
   - result, err := c.redisStore.ExecuteLuaScript(ctx.Request.Context(), peekResetTokenScript, []string{getResetTokenKey(token)}, []interface{}{})
   - if errors.Is(err, redis.Nil) || (err == nil && result == nil) -> return "", "", ErrNotFound
   - else if err != nil -> log (no token in the log) and return "", "", fmt.Errorf("%w: %v", ErrStoreUnavailable, err)
   - value, ok := result.(string); if !ok -> return "", "", fmt.Errorf("%w: invalid token value", ErrStoreUnavailable)
   - return parseTokenValue(value)
   - Do NOT use redisStore.Get for reset tokens anywhere.
   - Keep the existing logger fields (Class, Method). Do not log the token. Not-found is an expected outcome, so log it at Debug or not at all, not Error.

3. Do not change CreateResetToken, ConsumeResetToken, the flow methods, or the RedisStore interface.

TESTS
A. Update the existing PeekResetToken unit tests: they must mock ExecuteLuaScript (not Get).
   - found -> returns the tenantID and identityID
   - Lua returns nil, and also the redis.Nil error case -> ErrNotFound
   - other Redis error -> ErrStoreUnavailable (errors.Is), never ErrNotFound
   - result of a non-string type -> ErrStoreUnavailable
   - malformed value (no colon, or an empty part) -> ErrNotFound
B. Add a real-store integration test against miniredis. Reuse the miniRedisDb adapter in cache/password_lockout_cache_lua_test.go. Run the real RedisStore wrapper, not mocks:
   1. CreateResetToken("csr", "identity-1", ttl) -> ok
   2. PeekResetToken -> "csr", "identity-1", nil
   3. PeekResetToken again -> same result (Peek must not consume)
   4. ConsumeResetToken -> "csr", "identity-1", nil
   5. ConsumeResetToken again -> ErrNotFound
   6. PeekResetToken after consume -> ErrNotFound
   7. Fast-forward miniredis past the TTL -> Peek returns ErrNotFound
C. Add a real-store round-trip test for the flow too: Create -> Get (all fields equal) -> Update (change Resends) -> Get (Resends updated, TTL not extended) -> Delete -> Get returns ErrNotFound.

ACCEPTANCE
- go build, go vet and go test ./cache/... pass.
- The miniredis Create -> Peek -> Consume -> Consume(not found) test passes.
- PeekResetToken no longer calls redisStore.Get.
