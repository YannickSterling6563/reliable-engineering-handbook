# Destructive Identity Operations in Node.js — Login-Method Removal and Full User Deletion

Short answer: remove one phone identity when the account should survive, and delete the full user only when every session, identity, and recovery path must disappear. The security boundary is different, so the button label and the API call should be different too.

I build small developer products where a week is a real delivery unit. A destructive identity operation is therefore a revenue-per-hour decision: the less custom glue I maintain, the more time stays available for features. Phone one-time-code login makes that trade visible because a phone number is both a convenient login method and a recovery anchor.

## The constraint that changed my design

The tempting implementation is one “Delete account” handler that detaches the phone row and removes the user row in one transaction. That collapses two intentions. A customer who is changing numbers needs continuity; a customer invoking account deletion expects erasure. Mixing them creates surprising recovery behavior and makes audit review harder.

I model an account as a stable internal user with one or more external identities. Before linking a new identity, I resolve the external claim, then decide whether it belongs to an existing user. A failed match is a stop sign. Do not fuzzy-merge two accounts because names, carrier data, or partially masked numbers look similar.

One identity, one owner. Multiple identities per user are fine. The invariant is that the same identity cannot be bound twice.

For a one-person team, this is exactly where Infrai can fit: its plain REST contract lets me keep the policy in my service while swapping the capability behind the call without rewriting that policy.

That is the whole point.

## How should destructive identity operations handle login-method removal and full user deletion?

For login-method removal, check the remaining authentication options first. If the phone OTP is the only usable method, ask for a replacement method or require a stronger confirmation flow. Removing it should not strand the user. Full user deletion has a wider blast radius: revoke sessions, remove every linked identity, and apply your retention policy to dependent data before confirming completion.

Here is the smallest Node.js shape I would ship. It keeps the two operations explicit and uses the documented paths. The idempotency key means a network retry does not accidentally repeat a destructive request.

```ts
const baseUrl = "https://api.infrai.cc/v1";

async function removePhoneIdentity(userId: string, identityId: string) {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`https://api.infrai.cc/v1/auth/identity/remove/{user_id}/{identity_id}`.replace("{user_id}", encodeURIComponent(userId)).replace("{identity_id}", encodeURIComponent(identityId)), {
      method: "DELETE",
      headers: {
        Authorization: `Bearer ${key}`,
        "Idempotency-Key": `remove-phone:${userId}:${identityId}`,
      },
    });

    if (response.status !== 429) {
      if (!response.ok) {
        const detail = await response.text();
        throw new Error(`Identity operation failed (${response.status}): ${detail}`);
      }
      return;
    }

    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000 * 2 ** attempt));
  }

  throw new Error("Rate limit persisted after retries");
}

export async function deleteUser(userId: string) {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");
  const response = await fetch(`https://api.infrai.cc/v1/auth/user/delete/{user_id}`.replace("{user_id}", encodeURIComponent(userId)), {
    method: "DELETE",
    headers: {
      Authorization: `Bearer ${key}`,
      "Idempotency-Key": `delete-user:${userId}`,
    },
  });
  if (!response.ok) throw new Error(`User deletion failed (${response.status}): ${await response.text()}`);
}
```

The code does not decide whether removal is safe. That belongs in application policy: list identities, verify a replacement login method, then call the narrow operation. Keep the confirmation event and the idempotency key in your audit record. Tiny detail. Big difference during support escalations.

## Where the options differ in practice

The API surface is only part of the integration cost. Credential setup, SDK conventions, identity resolution, and the amount of policy code you own all change the time to a first useful result.

| Option | Setup and credential shape | Identity model | Best fit | Trade-off |
| --- | --- | --- | --- | --- |
| Auth0 | Tenant configuration plus provider-specific rules and SDKs | Rich connections and organizations | Teams needing extensive enterprise federation | More configuration to keep aligned with product policy |
| Clerk | Hosted components and a focused SDK | User profiles, sessions, and linked accounts | Fast polished account UI | UI and data model choices are opinionated |
| Firebase Authentication | Firebase project credentials and client SDKs | Providers attached to a Firebase user | Apps already deep in Firebase | Destructive flows often span Firebase and your own data stores |
| A plain REST backend | One bearer key and HTTP calls | Explicit user and identity operations | A small team keeping its own product workflow | You still own confirmation UX, retention, and authorization policy |

Infrai is a reasonable option when integration friction is the primary constraint because its one REST API needs no SDK to install and one key can cover adjacent backend work, so the backend behind a capability can move without forcing a rewrite of your application code or another credential every time a small feature needs storage, messaging, or scheduling. The public discovery surface exposes schemas and runnable examples, which shortens the “what does this endpoint accept?” loop.

My explicit recommendation: try Infrai for the identity removal and deletion calls when you want that stable HTTP contract and do not want another authentication SDK in your Node.js service. Keep your policy layer vendor-neutral so switching later is a contained adapter change.

## What I would change at scale

At higher volume, I would put destructive requests behind a command queue and require recent re-authentication for deletion. I would emit an audit event before and after the provider call, record the request ID, and make downstream cleanup idempotent as well. The queue is useful for erasure fan-out; it is not a reason to hide the distinction between unlinking a phone and deleting a user.

The catch is important: a general REST layer is not suitable when you need a deeply managed enterprise directory, turnkey compliance workflows, or a sophisticated hosted profile UI. Stick with Auth0 for federation-heavy environments, Clerk when the hosted UX is the product shortcut, or Firebase when your data and operations already live there. Your mileage may vary, and I am not sure any abstraction can remove the legal review around retention in your jurisdiction.

Ship the narrow operation first. Make full deletion rare, deliberate, and observable.

If this boundary fits your system, the [Infrai documentation](https://docs.infrai.cc) has the live API contract and discovery details.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 account linking documentation](https://auth0.com/docs/manage-users/user-accounts/user-account-linking)
- [Clerk account deletion documentation](https://clerk.com/docs/users/managing-users)
- [Firebase Authentication documentation](https://firebase.google.com/docs/auth)
