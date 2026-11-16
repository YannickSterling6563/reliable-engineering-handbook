# School DNS Record Writes With 3 Evidence Gates for Upsert Provisioning

| Write policy | Proposed default boundary | Reason to choose it |
| --- | --- | --- |
| Upsert | A dedicated verification record controlled by the application | Repeated requests can converge on the same intended value |
| Create | A name whose existing contents must be preserved | A collision should stop onboarding for review |
| Update | An existing record with an expected prior state | A change should require evidence that the target still matches |

Upsert reduces retry bookkeeping, but its useful boundary is narrower than an entire school domain. Choose it for an application-controlled verification record; keep create and guarded update explicit for records outside that boundary. Short answer: make scoped upsert the default provisioning intent in Node.js, then require separate ownership and mail-authentication evidence before education onboarding completes. A successful write alone shouldn't open that gate.

This decision note proposes 3 evidence gates: safe mutation, domain proof, and mail readiness. They are application policy, not a DNS standard. The goal is to let a one-person education SaaS ship weekly without turning every school signup into an investigation of somebody else's mail configuration.

## What should DNS record upsert prove before provisioning completes?

It proves less than the onboarding screen wants to say. In this design, a write acknowledgment records that the mutation adapter accepted the operation. The application still needs a separate observation of its verification challenge before it marks the school domain as proven.

Keep those states separate.

Consider an illustrative school domain, academy.example. The proposed challenge lives at _school-proof.academy.example; mail policy lives elsewhere. A request to provision the challenge should carry a stable desired value for that verification attempt. Retrying that request should converge on the same state, while an unexpected value should produce a conflict instead of an automatic replacement. The exact label, token lifetime, and challenge validation rules are application choices. None of them grants permission to modify unrelated TXT records.

Mail evidence answers another question. RFC 7489 describes DMARC using SPF or DKIM authentication together with identifier alignment. Observing a challenge doesn't establish those properties. Likewise, reading a DMARC policy doesn't demonstrate that a particular school notification authenticated and aligned.

That distinction matters for education onboarding: control of the requested domain and evidence about messages sent for that domain belong in different fields. A school administrator needs to know which step remains, without being told that a DNS mutation guarantees delivery.

## Criterion one is the evidence behind mail readiness

Use deliverability evidence as the decision axis, with a deliberately limited claim. Under RFC 7489, a DMARC pass requires an aligned authentication result from SPF or DKIM. DMARC policy and reporting provide evidence about authentication and requested handling; they don't promise inbox placement. The onboarding state should therefore say what was observed, such as an aligned authentication result for a test message, instead of claiming that all future messages will arrive.

For this proposed workflow, finishing onboarding requires both an observed ownership challenge and an acceptable mail-authentication observation. Keep the evidence attached to the sender identity and the verification attempt being evaluated. If the intended sender changes, evaluate that evidence again rather than carrying forward a generic green check. This is a proposed acceptance rule, not a requirement imposed by DMARC.

A policy record is useful context. It isn't the test message.

RFC 7489 also gives a concrete reason to avoid treating mail policy as a disposable provisioning value: multiple DMARC records at the policy lookup location cause policy discovery to terminate without applying DMARC processing. Blindly adding another record is therefore a poor recovery rule. If an onboarding request encounters existing mail policy, stop and evaluate the intended change. The convenient retry behavior of a dedicated challenge shouldn't decide how a school's mail policy is edited.

## Criterion two is the authority to replace existing state

Idempotency needs a definition.

Here it means that replaying the same provisioning intent converges on one intended state without accumulating extra changes. That definition is a contract for the application and its adapter; a method named upsert doesn't establish the contract by itself.

Use the narrowest target the workflow can own. The proposed identity for a verification operation includes its tenant, domain, record name, and verification attempt. Persist the desired challenge before attempting the write. A timeout should lead to observing that same intent, not generating a replacement challenge merely because the response was lost.

The difficult case is disagreement. If the observed challenge differs from the persisted value, treat it as a conflict that needs a new decision. If the adapter cannot protect a read-then-write sequence from concurrent changes, it cannot honestly promise a guarded replacement. Keep that limitation visible at the adapter boundary and leave onboarding pending. Serializing application jobs can reduce conflicts between those jobs, but it doesn't establish exclusive control over an external editor. For example, consider two illustrative requests using the same school domain but different verification attempts: completing the older request must not erase the newer challenge or mark the newer attempt proven. The persisted attempt identifier gives the application a boundary for rejecting that stale result; the adapter still needs its own mutation conflict protection.

For a solo founder, the revenue-per-hour lens favors a small state machine with clear evidence over a broad DNS editor. Outsource the undifferentiated execution through a narrow interface; retain the decisions about what may change and what counts as proof. That keeps the weekly shipping budget focused on learning workflows rather than a general-purpose control panel.

## One implementation separates mutation intent from admission

The following TypeScript example is a proposed orchestration contract. It assumes a persisted challenge and a tenant-scoped adapter that handles exactly one dedicated verification record. It doesn't implement a DNS provider API, claim atomicity, or alter mail policy.

```ts
type Challenge = Readonly<{
  tenantId: string;
  domain: string;
  recordName: string;
  attemptId: string;
  value: string;
}>;

type ProofResult = "matches" | "pending" | "conflict";
type MailResult = "aligned" | "pending" | "failed";

interface EvidencePorts {
  // Accept a persisted intent for an application-controlled record.
  ensureChallenge(challenge: Challenge): Promise<void>;
  observeChallenge(challenge: Challenge): Promise<ProofResult>;
  // Evaluate evidence for the intended sender and this attempt.
  observeMail(challenge: Challenge): Promise<MailResult>;
}

type Decision =
  | "review_conflict"
  | "await_domain_proof"
  | "await_mail_evidence"
  | "review_mail_failure"
  | "ready";

async function evaluateOnboarding(
  challenge: Challenge,
  ports: EvidencePorts,
): Promise<Decision> {
  await ports.ensureChallenge(challenge);
  const proof = await ports.observeChallenge(challenge);
  if (proof === "conflict") return "review_conflict";
  if (proof === "pending") return "await_domain_proof";

  const mail = await ports.observeMail(challenge);
  if (mail === "failed") return "review_mail_failure";
  if (mail === "pending") return "await_mail_evidence";
  return "ready";
}
```

Exceptions deliberately propagate. The job runner should record the failed attempt, preserve its intent, and retry according to a bounded policy. It shouldn't convert an observation error into either proof of absence or a ready decision. Before persisting ready, the surrounding application must also check that the attempt is still current; a result from an earlier attempt shouldn't admit a changed sender configuration.

Test the boundaries that could admit a school incorrectly: a write followed by pending proof, matching proof with pending mail evidence, and an obsolete attempt returning success after its replacement. Test a repeated intent and an unexpected existing value too. These checks earn their maintenance cost because each protects a specific admission decision.

Deploy the adapter behind that contract. Record the attempt identifier, mutation outcome, proof observation, and mail result separately, without logging challenge secrets or message contents. An operator should be able to distinguish a mutation failure from missing mail evidence without reconstructing the entire signup.

## When create or guarded update is the better choice

Create is the runner-up when any existing value should require review. It gives up automatic convergence on a preexisting record in exchange for an explicit collision decision. For a school-owned name with unclear editing authority, that's a useful trade.

Guarded update fits a deliberate change to a known record. Require the expected prior state and a way to reject conflicting changes; when the adapter cannot enforce that condition, route the operation to review. Neither choice should inherit the permissive replacement rules of an application-controlled challenge.

Use scoped upsert for repeatable challenge provisioning, create for protected absence, and guarded update for intentional replacement. Complete education onboarding only after the current attempt has both ownership proof and the required mail evidence. That's a decision rule small enough to maintain while shipping weekly.

## References

RFC 7489 supplies the mail-authentication, identifier-alignment, policy-discovery, and reporting basis discussed here. The challenge workflow and TypeScript interfaces are proposed application design.

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance, especially Sections 3.1, 6.6.3, 6.6.4, and 7: https://datatracker.ietf.org/doc/html/rfc7489

## Sources

- https://datatracker.ietf.org/doc/html/rfc7489
