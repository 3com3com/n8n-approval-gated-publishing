# Setup notes

Not a manual. The things that are not obvious, and the order to do them in.

---

## What you just imported

**A machine with no cargo.**

The wiring is finished and has run in production: research, draft, images, human approval, publish, threaded replies, and an alert either way. That part works.

**What it says is not in here.** Every topic, hashtag, and rule about what it must never claim is a placeholder. Import it, press run, and it produces something confident, well-formatted and worthless.

That is deliberate. The content is the half only you can write.

---

## Order

| | | Needs n8n? |
|---|---|---|
| 1 | Two minutes of safety | yes |
| 2 | Decide what it publishes | **no** — do it on paper |
| 3 | Credentials and variables | yes |
| 4 | Test before it can embarrass you | yes |
| 5 | One reminder before you walk away | no |

Step 2 is where people stall, and it is the one step that needs no tooling.

---

## 1. Two minutes, do it now

n8n saves node outputs in full. If a token ever passes through one, it sits in the execution log in plain text.

**`⋯` (next to Save) → Settings:**

| Field | Set it to |
|---|---|
| Save successful production executions | Do not save |
| Save failed production executions | **Save** — you want to debug failures |
| Save manual executions | Do not save |

No token passes through a node output in *this* workflow, so keeping failed-run data is safe and useful. In any workflow that *handles* a token — a refresh job, a rotator — turn all three off.

**⚠️ n8n does not apply a file's `settings` block on import.** These values are written in `workflow.json` and they will not take effect. Same for timezone. Set both by hand, every time you import anything.

---

## 2. Decide what it publishes

Answer these before opening a node. Nothing here is technical.

| | Question | Sets |
|---|---|---|
| a | What is the subject, in one sentence a stranger understands? | `NICHE` |
| b | Two content pillars — related, not identical | `TOPIC_A` / `TOPIC_B` |
| c | Four post angles under each | node `Generate Topic1` |
| d | What must it never claim? | `banned` in `Format + Safety Gate` |
| e | The line appended to every post | `DISCLAIMER_PRIMARY` |

**(c) is the one that decides quality.** Each entry has an `angle` field — that is what the model writes from.

- `"grind size"` → filler
- `"why a finer grind is not always stronger coffee"` → a post

**(d) is the one that protects you.** The test: *would this embarrass me, or get my account restricted, if the model wrote it at 3am with nobody watching?* Health claims, income promises, unsupportable superlatives, competitors by name, whatever your regulator cares about.

It is cheap to write and it is the only thing between a hallucination and your account. **Write it before the first live run, not after.**

---

## Do this part with an AI

Faster, and it is what the tool is for.

**Attach `workflow.json` and this file** to ChatGPT / Claude, then paste:

```
I've imported an n8n workflow that publishes to social media on a
schedule, with a human approval step. Workflow file and setup notes
attached.

I'm setting it up for: [your business, 2-3 sentences]

Act as my setup partner. Work through the notes with me one step at
a time, don't dump everything at once.

Start with step 2 - deciding what it publishes. Ask me questions
until we have a NICHE, two pillars, four angles under each, a
banned-phrase list, and a disclaimer. Push back if I'm vague;
"business tips" is not a niche.

Then help me work through the rest.
```

Keep that conversation open for the remaining steps. Screenshot what confuses you; paste errors you hit.

**Never paste a token, API key or password into a chat.** Paste the error, not the credential. If the error contains a long random string, cut it out first.

---

## 3. Credentials and variables

**Seven services:** Anthropic, OpenAI, Perplexity, Telegram, Threads, Instagram, Cloudinary. Six become n8n credentials. Cloudinary is not one — the three `Upload to Cloudinary` nodes read `CLOUDINARY_CLOUD_NAME` and `CLOUDINARY_UPLOAD_PRESET` from variables instead.

⚠️ **An unsigned Cloudinary upload preset is a secret.** Cloud name plus unsigned preset is all anyone needs to upload into your account — no authentication involved. Storage burn and hostile content are the realistic outcomes. Use a *signed* preset, or if you keep it unsigned, restrict allowed formats, max file size and target folder. Never commit either value; that is why they are variables here.

Node names say which is which. Variable names are in the nodes that use them.

**Lock the approval webhook.** The `Webhook` node ships with Header Auth on and needs a credential to check against — a header name and a long random value you invent. Then make sure whatever sends your approval actually sends that header; a link clicked from a chat message cannot carry one. Either put something in between, or accept that the `uuid` is your only lock. **Decide that on purpose.**

**Three things that will cost you an hour each:**

- **n8n cannot change a credential's auth type.** Header Auth → Query Auth means a new credential and repointing every node. Miss one and you get a partial failure that looks like a platform bug. The 🔗 count next to the old credential is how many nodes still use it. Get it to zero.
- **Meta's refresh endpoints only read the token from the query string.** `Authorization: Bearer` returns `The parameter access_token is required. (code 100)` — which reads like the token is missing when it is right there.
- **n8n blocks `$env` inside Code nodes by default.** Normal fields read it fine. Two Code nodes here read `$env`; both fall back to defaults rather than crash, so it runs either way — you just get placeholder hashtags and slot labels.

---

## 4. Test before it can embarrass you

`dry_run` ships as **true**. Leave it.

1. Execute manually. Every node green.
2. **Read what it wrote.** Not "did it run" — read the text. Placeholder prompts produce plausible nonsense, and plausible is easy to mistake for correct.
3. Iterate until you would be happy to post it.
4. Set `dry_run` to false. Publish exactly one.
5. **Open the account and look at it.**

**Step 5 is not optional.** A green execution is not proof a post exists. That is the entire subject of this repository — see *The 16 days* in the README.

---

## 5. Before you walk away

Calendar reminder, **50 days from today**: *"check publishing tokens."*

Meta's long-lived tokens die at 60 days and **cannot be refreshed once expired.** Miss it and you redo the login flow by hand. Better: build the daily refresh job described in the README.

---

## 6. The error workflow

`error-workflow.json` is a separate three-node workflow: Error Trigger → build the message → send it to Telegram. It fires on **any** failed execution of the main pipeline and tells you the workflow name, the node that failed, the execution id and the error text.

1. Import `error-workflow.json` as its own workflow. You do **not** need to publish or activate it — n8n's docs: *"If a workflow uses the Error Trigger node, you don't have to publish the workflow."*
2. Open the main pipeline → `⋯` → **Settings** → **Error Workflow** → pick it.
3. It reuses `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID`. Nothing new to configure.

**⚠️ Step 2 does not travel with the file.** The `settings` block is not applied on import — same trap as the timezone and execution-data settings in step 1. Set it by hand.

**⚠️ You cannot test this with a manual run.** n8n's own docs are explicit: *"You can't test error workflows when running workflows manually. The Error Trigger only runs when an automatic workflow errors."* To test it, activate the main workflow, break one node's URL, and wait for a scheduled run — or fire the production webhook. A manual run will fail silently and you will conclude the alert is broken when it is not.

**Why the alert handles two shapes.** The Error Trigger emits `execution.*` when a node fails mid-run, and `trigger.*` — with no `execution` object at all — when the trigger itself dies. An alert that only reads `execution` goes quiet in exactly the case you most need it: the schedule stopped firing. This one reads both.

**What it does not do.** It reports failures. It cannot report a *run that never happened* — a disabled trigger, a paused workflow, an expired credential in a half nobody triggers. That is a different problem and it is the one that cost me sixteen days. See *The 16 days* in the README.

### Why the four publish nodes do not retry

Every outbound request in `workflow.json` ships with **Retry On Fail** on, **Max. Tries 3**, **Wait Between Tries 5000 ms** — except `Threads Publish`, `Threads Publish Reply 1`, `Threads Publish Reply 2` and `IG Publish Carousel`, which have retry off.

A request can time out *after* the platform has already accepted it. Retrying that is a second post. A container that gets created twice is wasted quota; a post that goes out twice is on someone's feed. **Retry the cheap half, fail loudly on the half you cannot undo.**

---

## When it breaks

1. Open the failed execution, click the red node, read the **Output** tab. n8n's message is a wrapper; the real error is inside.
2. **A Meta node failing with a useless message is almost always an expired token** — an expired Threads token returns an empty HTTP 500 with no body. Check the token's age before anything else.
3. Still stuck? Paste the error into that AI conversation, with any long random strings removed.
