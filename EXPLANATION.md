# Bug Explanation

## What was the bug?

The `Client.request()` method failed to refresh OAuth2 tokens when `oauth2_token` was a dictionary instead of an `OAuth2Token` instance. The test `test_api_request_refreshes_when_token_is_dict` expected the Authorization header to be set after refresh, but it was `None`.

## Why did it happen?

The refresh condition on line 30 was:
```python
if not self.oauth2_token or (isinstance(self.oauth2_token, OAuth2Token) and self.oauth2_token.expired):
```

This logic only refreshed when:
1. Token was falsy (`None`), OR
2. Token was an `OAuth2Token` instance AND expired

When `oauth2_token` was a dict:
- `not self.oauth2_token` evaluated to `False` (dicts are truthy)
- `isinstance(self.oauth2_token, OAuth2Token)` evaluated to `False` (dict ≠ OAuth2Token)
- The entire condition evaluated to `False`, skipping refresh
- The subsequent `isinstance` check on line 34 also failed, so no Authorization header was set

## Why does the fix solve it?

The fix changes the condition to:
```python
if not self.oauth2_token or not isinstance(self.oauth2_token, OAuth2Token) or self.oauth2_token.expired:
```

Now it refreshes when the token is NOT an `OAuth2Token` instance (including dicts), ensuring proper token objects are always used.

## Edge case not covered

**Race condition with token expiration**: If a token expires between the `expired` check and the actual request execution (e.g., due to system clock adjustments or network delays), the request would proceed with an expired token. A more robust solution would check expiration with a safety margin (e.g., refresh if expiring within 60 seconds).
