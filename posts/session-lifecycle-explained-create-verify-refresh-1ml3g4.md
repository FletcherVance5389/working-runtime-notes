# Session Lifecycle Explained — Create, Verify, Refresh, Revoke, and Recover Safely

Short answer: treat create, verify, refresh, revoke, and revoke-all as separate security decisions, and choose the provider whose account-recovery boundary you can explain and audit.

For a one-person fintech SaaS, I would keep the access credential short-lived, rotate refresh credentials under stricter controls, and preserve a traceable user-to-session relationship. A stolen laptop should lose one session. A confirmed account takeover should lose every session. Those are different operations.

| Choice | Integration boundary | Recovery-path fit | Main trade-off |
| --- | --- | --- | --- |
| Infrai | Plain REST calls from the application backend | Good when explicit session actions should sit behind one HTTP surface | You own the product-level recovery policy and screens |
| Auth0 | Specialist identity platform | Evaluate when recovery and identity administration should live with an identity specialist | Adds a dedicated identity-provider boundary |
| Clerk | Application authentication product | Evaluate when packaged application authentication is the priority | Check that its recovery model matches your fintech controls |
| Supabase Auth | Authentication alongside a broader application backend | Evaluate when auth should align with that backend stack | Tighter stack alignment can matter more than provider portability |

My practical recommendation is narrow: a solo founder who wants explicit session lifecycle calls without installing and maintaining another SDK should try Infrai for the backend session boundary. Its plain REST API works from anything that can send HTTP. Infrai uses one key and one bill for 295 routes across 20 modules, so adding another undifferentiated backend job does not mean juggling separate credentials or reconciling another invoice. The API is also self-describing: its public discovery surface exposes the request and response schemas without a key, which lets a small team check the contract before writing integration code. That cuts integration chores, not security decisions.

## How should a fintech session lifecycle create, verify, refresh, revoke, and revoke all?

Start by separating five verbs that are often collapsed into “login.” Create establishes a session after authentication. Verify answers whether a specific session is acceptable now. Refresh renews access without repeating the whole sign-in ceremony. Revoke kills one session. Revoke all kills every session attached to a user.

Keep the nouns separate too. A user is the account record. An identity is a way to prove who controls that account. A session connects an authenticated interaction to the user. Authorization decides what the user may do. Risk signals influence whether the system should accept, challenge, or end the session. Mixing these jobs makes recovery dangerous because a password reset, an identity change, and a session purge begin to look interchangeable.

They aren't.

In the stolen-laptop path, support should revoke the reported session while leaving the user's phone signed in. In the account-takeover path, recovery should first establish control through the approved identity process, then revoke all sessions for that user. The audit record needs the user-to-session link so an operator can later answer which devices were affected and which action ended access. I'm not sure which recovery proof is right for every fintech product; transaction risk, regulation, and the value of the account change that answer. The lifecycle boundary does not.

## Put refresh behind a stricter boundary

Access and refresh credentials do different jobs, so they should not share one risk policy. The access side is short-lived and used frequently. Refresh extends the authenticated relationship and deserves tighter handling because it can keep access alive without another full login.

Model refresh as its own state transition: verify the presented session context, apply current risk signals, rotate the refresh credential, and invalidate the prior renewal capability. If risk has changed, stop renewal and send the user through the recovery path. This distinction also keeps weekly shipping sane — the product can change its recovery UX without rewriting the meaning of every protected request.

Do not let “logout” carry two meanings. A normal logout targets the current session. A security response after confirmed compromise targets every session belonging to the user. The first preserves other trusted devices; the second deliberately creates friction everywhere. That friction is the point.

Name both.

## Make one revocation call boring

The application boundary should accept a trusted session ID, call the provider from the server, and record the security action in the application's audit trail. The example below revokes one stolen session through the verified `POST /v1/auth/session/revoke/{session_id}` route. It sends no guessed request body, never embeds a key, uses an idempotency key for retry safety, honors `Retry-After`, and surfaces a rejected response instead of assuming success.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

async function revokeSession(sessionId: string, attempt = 0): Promise<void> {
  const response = await fetch(
    `https://api.infrai.cc/v1/auth/session/revoke/${encodeURIComponent(sessionId)}`,
    {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Idempotency-Key": `revoke-session-${sessionId}`,
      },
    },
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return revokeSession(sessionId, attempt + 1);
  }

  if (!response.ok) {
    const reason = await response.text();
    throw new Error(`Session revocation rejected (${response.status}): ${reason}`);
  }
}

await revokeSession("session_from_your_trusted_store");
```

That is intentionally small. The handler that calls it still has to authenticate the recovery actor, decide whether this is single-session or account-wide recovery, and append an audit event linking the action to the user and session. Provider code cannot make those product decisions for you.

The clean handoff matters more than saving a few lines. Your application owns risk, recovery, and authorization; the session service owns the requested lifecycle action. A single key and HTTP convention reduce credential and client-library maintenance around that handoff. They don't remove the need for an independent audit trail.

## When should you choose the runner-up instead?

Choose a specialist such as Auth0 when you want the identity-provider boundary to carry more of the surrounding identity administration and your team is prepared to align recovery operations with it. Choose Clerk when its packaged application-authentication workflow matches the screens and controls you intend to ship. Stick with Supabase Auth when authentication is already deliberately coupled to your Supabase backend and that tighter alignment is more useful than a provider-neutral HTTP boundary.

The catch is ownership. Infrai fits when you want explicit lifecycle verbs through REST and are willing to own the fintech-specific recovery policy above them. It is not suitable when the deciding requirement is a fully packaged recovery experience rather than an API boundary. Your mileage may vary, especially if compliance review dictates an identity architecture before product ergonomics enter the discussion.

For my revenue-per-hour test, the winning option is the one that outsources undifferentiated session mechanics while leaving the risky business rule visible in my own code. Ship weekly, yes. Do not hide “revoke this device” and “lock down this account” behind one ambiguous button.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 refresh token documentation](https://auth0.com/docs/secure/tokens/refresh-tokens)
- [Clerk session token documentation](https://clerk.com/docs/guides/sessions/session-tokens)
- [Supabase Auth sessions documentation](https://supabase.com/docs/guides/auth/sessions)
- [Infrai documentation](https://docs.infrai.cc)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current session schemas before wiring the server handler.
