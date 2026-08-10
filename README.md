# n8n Approval-Gated Publishing Pipeline

An n8n workflow that drafts a post with an LLM, generates its own images, waits for a human to approve it, then publishes and threads its own follow-up replies.

**Running in production since 2026-03-28.**

| | |
|---|---|
| Days in operation | **133** |
| Generation schedule | **3 runs/day**, ~2 min each |
| **Longest undetected breakage** | **16 days** |
| Time to recover once found | **46 minutes** |
| Publish path last verified | 2026-08-08 |
| Size | 63 nodes · 6 timed waits · 2 platforms |

I have also lost runs to LLM billing lapses and **did not record how many.** "133 days, zero downtime" would be a better sentence and it would not be true.

---

## The 16 days

**Green executions told me nothing about whether publishing worked.**

This runs as two separate executions:

```
Schedule, 3x/day    -> research -> draft -> images -> hold     [half 1]
Webhook, on approve -> create -> wait -> publish -> replies    [half 2]
```

Half 1 runs on a timer. Half 2 runs only when a human approves.

I went a while without approving. Half 1 stayed green the whole time — it never calls a publishing API, so it had nothing to fail on. Meanwhile the platform's long-lived token expired (issued 05-23, dead 07-22). **Half 1 did not notice, because half 1 does not use that token.** Neither did I.

I found out on 08-07, when I finally clicked approve:

```
06:36:12  Error in 1.985s   Create Container
07:03:50  Error in 499ms    retry
07:18:25  Error in 48.002s  retry
07:22:00  Succeeded in 1m 53.112s
```

Sixteen days of breakage, discovered by accident.

**A monitored path and an unmonitored path do not average out.** An approval gate makes this worse, not better: the gate is exactly what stops the risky half from running, so the risky half is exactly what never gets exercised.

**Fixed by** two daily jobs that refresh every publishing credential unconditionally every 7 days, validate the new token before writing it back, and report **on quiet days too** — because a job that only speaks when it acts cannot tell you the scheduler died. Verified 08-08 through the do-nothing branch.

Why unconditional rather than a threshold: the only date the API exposes is the credential's `updatedAt`, which also moves when you rename it. It is not the token's issue date. A fixed cadence you can trust beats a threshold on a signal that lies.

---

## Six things the docs don't tell you

**1. An expired Threads token returns an empty HTTP 500.** No OAuthException, no 401, no `code: 190` — an empty 500 with no body. Instagram, same expiry, returns a clean `code: 190`. If a Meta node fails and tells you nothing, check the token's age first.

**2. Instagram's refresh endpoint only reads the token from the query string.** Send `Authorization: Bearer` and you get `The parameter access_token is required. (code 100)` — an error that reads like the token is missing when it is right there in the header.

**3. n8n cannot change a credential's auth type.** Header Auth to Query Auth means a new credential and repointing every node. Miss one and you get a partial failure that looks like a platform problem.

**4. A field name starting with `=` silently becomes a different field.** `=slot_mode` is not `slot_mode`. Downstream, `$json.slot_mode` is `undefined`, and the ternary reading it always takes its else branch. Mine did that for 133 days — the pipeline never failed, it simply stopped varying content by time slot. *"Produced a post"* and *"produced the right post"* look identical from outside. **Search your workflows for `"name": "="`. Ten seconds.**

**5. A node with no incoming connection is invisible on a canvas this size.** Mine carried two X nodes wired to each other and to nothing else. Never ran, not once in 133 days. Reading the JSON never showed it — they connect to *each other*, so every orphan check said they were fine. I found them by importing into a clean instance and seeing them float above the chain. They are not in this repo.

**6. n8n does not apply a workflow's `settings` block on import.** Timezone and execution-data retention fall back to instance defaults even though the values are in the file you just imported. A workflow of mine said `saveDataSuccessExecution: none` and had been storing every execution since import — including node outputs containing tokens.

**[`docs/after-import.md`](docs/after-import.md) is the ten-minute checklist that closes all of these.** Read it before you run anything.

---

## Why there are six wait steps

The publishing API returns a container ID **before the post exists**. Publish immediately and the call succeeds — HTTP 200, green execution, nothing posted. No error raised, so nothing alerts.

That is the failure mode this is built around: **it succeeds and does nothing.** One wait between every create and its publish; six in total. The pipeline then reports on success *and* failure — a pipeline that only speaks when it breaks cannot tell you it has stopped.

---

## Architecture

```
[ SCHEDULE ]  3x/day
   |- trend research -> draft -> translate -> structure    LLM
   |- image prompts -> 3 images -> CDN upload              persistent URLs,
   |                                                       not the model's expiring ones
   |- safety gate         banned-phrase check, both languages
   |- IF safe? --no--> notify blocked, stop
   |_ hold for approval, state persisted

[ WEBHOOK ]  on human decision - header auth required
   |- IF approved? --no--> notify rejected, stop cleanly
   |- create container     post is NOT live yet
   |- wait                 <- the whole point
   |- publish -> reply 1 (create->wait->publish) -> reply 2
   |- platform 2: 3 containers -> waits -> carousel -> publish
   |_ append publish log -> notify, success AND failure
```

---

## What this is

**A reference implementation, not a product you install.** Seven external services, and every piece of domain content — topics, hashtags, voice, disclaimers — is a placeholder. It will run and produce nothing worth publishing until you configure it. Ships with `dry_run: true`.

- Reading it? Start at **The 16 days**.
- Running it? **[`docs/after-import.md`](docs/after-import.md)** first. Credentials and variables are listed there.

`workflow.json` — credentials stripped, prompts parameterised. Import it, fill the variables, run it. Nothing is locked to me.

`error-workflow.json` — a three-node workflow that catches any failed execution of the pipeline and sends the failing node and the error text to Telegram. Import it too and set it as this workflow's Error Workflow. Details in [`docs/after-import.md`](docs/after-import.md).

---

## Security

- **The approval webhook ships with header auth on.** Mine ran without it for 133 days. I was thinking about what the gate protects, not about what protects the gate.
- **A disabled node is a disabled alarm.** My safety-gate notification was switched off. The gate still blocked content; it just stopped telling me. It ships enabled here.
- **The credential that writes credentials.** The refresh jobs use an n8n API key that can update every credential on the instance. Scope it to `credential:read,update,list` and know where it lives.
- **Execution data is where secrets sit.** If a token passes through a request body, disable execution saving on that workflow — and re-check after every import, because the setting does not travel with the file.
- **Retries are not free on an irreversible call.** Every outbound request in here ships with Retry On Fail, Max. Tries 3, Wait Between Tries 5000 ms — except the four that actually publish, which have retry off. A publish that times out after the platform already accepted it would post twice on retry, and a published post cannot be recalled. Those four fail loudly instead, and the error workflow tells you.
- **Known and not fixed:** the publish path still has no independent end-to-end health check. The daily refresh exercises the credential — which is what caught the failure above — but does not prove a post would succeed. A real canary would publish to a private target on a schedule. I have not built that.

---

## When NOT to use this

- **You need instant publishing.** Two minutes per run is the price of not failing silently.
- **You need guaranteed delivery.** Retries and alerts are not exactly-once.
- **Your platform has no create/publish split.** Then half of this is overhead.
- **Your LLM quota is tight.** One run makes many model calls. It has been rate-limited before.
- **You want set-and-forget.** Read *The 16 days* again.

---

Configuring this for a specific business — seven credentials, your topics, your approval channel — is the part that takes time. Contact is in the profile.

If you have run an approval gate in production longer than this and found a failure mode I have not, open an issue. I would rather read about it than discover it.
