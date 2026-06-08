---
title: "Shipd"
description: "$250"
icon: "article"
date: "2026-06-08"
lastmod: "2026-06-08"
draft: false
toc: true
weight: 1
---

everything i wish someone had told me before my first submission - the workflow, the caveats, the platform bugs, and how to not torch all your energy on a single mistake. these are casual notes from actually grinding the quest, not official docs. if something here is wrong or gets patched, ping me and i'll update.

{{< figure src="p1.png" alt="shipd cipher" >}}

cipher is shipd's challenge platform where you build security scenarios for AI agents to solve and get paid when they're approved. The important word here is *scenarios*. This isn't a CTF, nobody is hiding a `flag{something}` in a PNG, and if your challenge description sounds like it was written by a sleep-deprived junkie, it's probably headed for rejection.

the mental model: you ship a service that's quietly broken in some real-world way, an agent has to probe it, figure out the weakness, and exploit it, and a verifier decides whether it actually pulled it off.

three things you're really shipping:

- **environment** - a docker container running your (vulnerable) service
- **reference solution** - your own working exploit. this is what proves the challenge is actually solvable
- **verifier** - a script that grades the agent's output and writes a reward, 1 or 0

if you keep those three straight in your head, everything else is detail.

## the tabs (the lifecycle of a challenge)

{{< figure src="p2.png" alt="the task, harbor and checks tabs" >}}

you move left to right:

1. **task** - title, category, and the agent prompt (the task the agent sees). written in a rich-text editor.
2. **harbor** - the actual bundle: your `environment/`, the verifier, the reference solution, and `task.toml`. this is where you upload zips and hit *validate*.
3. **checks** - the killzone. setup checks → environment checks → agent runs (the screening) → submission gate.

two things to burn into your brain early:

- **everything goes stale on any change.** touch the prompt? gates re-stale. re-upload the env? re-stale. you will re-run checks a lot, and each run costs energy, so batch your changes.
- **energy regenerates slowly.** the screening is the expensive run. don't spend it on a challenge you haven't sanity-checked yourself. we get 3/hour.

## the harbor bundle (what you actually ship)

{{< figure src="p3.png" alt="the harbor bundle structure" >}}

you upload the pieces as zips on the files tab, then hit **validate package**. validation is mostly structural + a build; the actual "does the reference solve it" check happens in the **environment checks** later.

`instruction.md` and `task.toml` are **generated** from the overview tab - you don't hand-write them, you edit the prompt/title and they regenerate.

btw you can also follow this structure and push it to github and import it directly into shipd - they support it!

#### task.toml (the two lines that make or break you)

```toml
[agent]
user = "player"

[verifier]
user = "root"
```

- **agent runs as `player`** (unprivileged). this is the user the solving agent - and the oracle check - run as. it must NOT be root, or the whole "the agent can't read the secret/source" premise falls apart.
- **verifier runs as `root`** so it can read root-owned reference files / the real secret to grade against.

other fields you'll set: `difficulty = "hard"`, `workdir = "/app"`, `allow_internet = false`, `category`.

⚠️ **footgun:** every time you re-upload the environment zip and re-validate, the platform tends to **revert `[agent] user` back to root and drop the `[verifier]` block**. re-add both, save, re-validate. check this every single time after a re-upload. it has bitten basically everyone.

### THE big caveat - the `/tmp/cipher-baseline` source leak

this is the one that makes people stare at their screen for an hour. read it carefully.

the harness **snapshots your workdir before the agent runs** (and it extends to any directory you name in `task.toml`), drops it as a `.tar` at **`/tmp/cipher-baseline`**, and uses it later to diff what the agent changed. you'll literally see it in the verifier wrapper output:

```
=== Cipher Harbor verifier wrapper ===
--- Capturing agent diff before verifier applies tests ---
```

so far so good - that tar is a verifier-side convenience for computing the agent's diff. **the problem:** the snapshot includes all your root-owned files *with their contents*, but the perms you so carefully `chmod`'d are **not enforced on the copy in `/tmp`**. so the agent, running as `player`, just reads your source straight out of the baseline tar.

in my case the tar happily exposed the source (`EPHEMERAL_BITS = 245`) - everything an agent needs to skip the intended attack. note: a secret generated **at runtime** (e.g. `os.urandom`) is safe, because it never touches disk at snapshot time. it's the **source files and anything written at build time** that leak.

{{< figure src="p4.png" alt="the cipher-baseline tar leaking the source" >}}

this applies to basically any challenge on the platform, so it's worth knowing cold.

**how to not get bitten:**

- keep your **source and any sensitive files OUT of the workdir.** put them in a root-owned, `700` directory like `/opt/yourtask/`, and keep `workdir = "/app"` as a throwaway the agent is welcome to poke at.
- the agent only ever needs `/app` (or wherever it writes its output). it does not need your source.
- other options people floated: `rm`/lock down `/tmp` before the agent starts, or set `/tmp` perms so `player` can't read the baseline. the cleanest fix is just **don't let anything secret live in the snapshotted dir in the first place.**

it's a genuinely peculiar bug and the right move is to point it out to the admins so nobody else loses an afternoon to it - but until it's patched, design around it.

## the checks pipeline

{{< figure src="p5.png" alt="the checks pipeline" >}}

### setup checks (~13)

the cheap, fast gate. **no code runs here** - it's almost entirely about your prompt and packaging being clean. the full 13:

- **Source Metadata** - required fields (title, category, etc.) present and well-formed. passes trivially if you filled the form.
- **Category Alignment** - your prompt actually matches the category you picked. a "crypto" task that reads like web gets dinged.
- **Text Validity** - the prompt text is well-formed and renders. catches broken markup / weird editor artifacts.
- **Prompt Word Count** - not too short, not a wall of text. *(stuck point: a 2-line prompt fails - give the agent enough to orient without handing it the attack.)*
- **Prompt Conciseness** - the flip side, don't ramble. tight and purposeful.
- **Prompt Hygiene** - no leftover junk: no flags, no TODOs, no internal notes, no accidental giveaway of the vuln. *(stuck point: a stray hint about the attack. keep it need-to-know.)*
- **Originality** - not a near-copy of an existing challenge or the sample. *(stuck point: people reskin the [badger-merkle](https://shipd.ai/quests/cipher/challenges/s57fdqd582n93qr18fbaktc201881svr?step=overview&example=badger-merkle-digest) example too closely - use it for tone, not as a template.)*
- **AI Trace Cleanup** - the "is this prompt AI-written" check, surfaced as the **AI-like %** badge. (the big one - callout below.)
- **Task & Verifier Quality** - an overall quality bar on the task + verifier. vague tasks / lazy verifiers get flagged.
- **Verifier Harness** - your `test.sh` is structurally sane: it runs, and writes the reward to the right place.
- **Verifier Bundle** - the verifier zip has the files it should, laid out right.
- **Verifier Alignment** - the verifier actually checks **what the task asks for.** prompt says "recover the secret" but the verifier checks something unrelated → fails.
- **Verifier Gameability** - can the verifier be tricked into passing without the real work? (burned a lot of people early - callout below.)

{{% alert context="info" %}}
**the AI-like % trap.** the badge is a **cached** score from the last analysis run. it does NOT recompute when you edit the prompt - i've reloaded the page with a fully humanized prompt sitting there and it still read 100% because the cache hadn't refreshed. it updates when setup checks re-run, or when you make a *real* edit in the editor (a scripted/programmatic paste sometimes won't trigger the re-score; real typing/pasting does). so if you humanized it and it still says 100%, don't panic - re-run setup. to actually drive the score down: write first-person, kill the balanced "on one hand / on the other" AI cadence, use contractions, cut the em-dashes, make it read like a person who's annoyed at a service wrote it.
{{% /alert %}}

{{% alert context="info" %}}
**verifier gameability (the early-days killer).** this gate hunts for verifiers that pass on a fake/trivial input. two classic mistakes: (1) the verifier **calls the live service** to grade - but the agent has enough access to poke that service, so it can be coaxed into a pass; (2) the verifier does a **transparent check** the agent can satisfy directly (a string match, or a `pow(sig, e, n) == message` style "decode" check). fix: make it **self-contained and opaque** - verify the *real* property with a real library (e.g. `pub.verify(sig, msg, ...)` for a standard signature, or a constant-time compare against the in-memory secret), and keep any non-standard tricks on the attack side, never the grading side. if a dumb input can pass, this check finds it.
{{% /alert %}}

### environment checks (~7)

the meaty, slow gate - it actually **builds your image and runs your exploit end to end.** it'll report the image size (e.g. `60 MB`) and `Built`. the full 7:
- **Environment Dockerfile** - your Dockerfile is valid and the build steps are sane. *(stuck point: missing apt/pip deps, or `COPY`ing files that aren't there.)*
- **Verify Build** - the image actually builds, reproducibly. *(stuck point: unpinned deps that build today and break tomorrow - pin your versions.)*
- **Reference Solution** - your reference exploit is present and wired to the solve path the oracle will run.
- **No-op Check** - run the verifier on an **untouched** image, no solve applied → must reward **0**. proves the task doesn't pass for free. *(stuck point: a verifier that defaults to pass, or an answer artifact already baked into the image → instant reward leak. default reward to 0, only flip to 1 on a real pass.)*
- **Oracle Check** - run **your reference solution, then the verifier** → must reward **1**. the "is it actually solvable + does the verifier honor a real solve" check. (biggest stuck point - callout below.)
- **Task Realism** - believable real-world-shaped task, or a toy? *(stuck point: source base too small / too contrived → flagged. callout below.)*
- **Reference Quality** - is the reference a legit demonstration of the intended attack, or does it cheese the verifier? a reference that takes a shortcut the agents can't, or that exploits a verifier hole, fails here.

{{% alert context="info" %}}
**the Oracle Check - write a LITERAL `0`/`1`.** the reward has to be an explicit `0` or `1` in `/logs/verifier/reward.txt`. **do not** write it through an unexpanded variable / env (`echo "$REWARD"` when `$REWARD` is empty, or `echo $SOMETHING`) - if it lands blank or as a non-`0/1` string, the oracle reads a fail and you'll swear your solve worked. it did - the reward *write* didn't. hardcode `echo 1 > /logs/verifier/reward.txt` on the success path.
{{% /alert %}}

{{% alert context="info" %}}
**Task Realism - "your source is too small."** common early rejection: the codebase is too thin / too obviously a puzzle. build out a real-ish service - a few endpoints, some plausible scaffolding (config, a couple of routes, a reason it exists: a KMS, a notary, a license server) so it reads like something that could ship in production. realism is *graded*, not vibes - a 30-line toy gets flagged.
{{% /alert %}}

### agent runs (the screening)

{{< figure src="p6.png" alt="agent runs, extreme" >}}

- even slower (**extreme**) 
- this is where agent actually tries to solve it. it's the expensive run. you're trying to land in the **hard-but-solvable** band (more on the bar below).
- nova screening is the full-eval run consistent of 3 nova agents and 3 nova-plus agents, it's the objective mainly is to make sure atleast one nova agent is able to solve it (*solvability*) and at max only 2 agents are solving it (*difficulty*).
- **read the trajectories**, not just the score. this is the part people skip and regret. open the per-run logs and actually read **how** each agent solved or failed. you're checking three things: (1) the solvers used the **intended attack**, not a cheese; (2) the failers failed for a **fair** reason - it's genuinely hard - not a blocker or a missing hint; (3) nobody's quietly reading your source out of the baseline tar ^^. the count can look perfectly fine while the *method* is all wrong - i've had a "solve" that was just an agent stumbling onto something i left open, which is a fix, not a win.
- **don't spend it blind** - run the challenge yourself locally and make sure ideally the shadow run is either a time-out or solves with a longer time (*~40+ minutes*). a screening that dies on a dumb env bug is ~12 energy straight down the drain.

### submission gate

{{< figure src="p7.png" alt="the submission gate" width="280" >}}

- easiest out of the bunch
- reap the fruits of your hardwork!
- will judge the solves of the agents and check if they are intended and all that. still costs 6 tokens.

## the ~~agents~~ clankers

{{< figure src="p8.png" alt="the solving agents" width="450" >}}

there are tiers of solving agents. practically:

- **Nova** is the screening tier - this is the one whose pass-rate decides "too easy" vs "good."
- **Orion** is another tier in the mix.
- **Vega** is the strongest, used as the feasibility check - if Orion can solve it, the challenge is provably solvable.

what they run underneath isn't officially documented - from what people have pieced together, nova looks like it's running `gemini-cli`, and the others are probably running the claude models. treat that as community guesswork, not gospel. **what actually matters is the bar, not the model.**

**the difficulty bar (this is the whole game):**

- you want it **hard but solvable.** Orion should be able to clear it; Nova should mostly *fail*.
- as a rule of thumb, you're aiming for **Nova solving at most ~2 out of 6 runs.** more than that and it gets flagged "too easy."
- **0/6 is usually fine** as long as the reference + Orion prove it's solvable - feasibility carries it. the fatal failure mode is *too easy*, not *too hard*.

so when you tune difficulty, err toward harder, as long as your reference still reliably solves it within the environment-check time limit.

## energy/tokens - seriously, don't burn them

- energy regenerates **slowly**, and every gate run costs some. rough ballpark from what i've seen: setup is cheap (fractions of a point), environment is several points, the screening is the big one (~12).
- **every change re-stales the gates**, so a "quick tweak" can cost you a full re-run of setup + env + screening. plan changes in batches.
- the runs themselves are **slow** (the screening especially - real agents solving in real sandboxes takes a while). go do something else and check back.

{{< figure src="p9.png" alt="a screening run approaching three hours" >}}

{{% alert context="info" %}}
here's a casual one approaching the 3 hour mark, highest was close to ~5 hours - so be patient and work on another problem - then again you risk running out a tokens - as you can see it's a slow game.
{{% /alert %}}

- sanity-check your challenge **locally** (run the service + your reference yourself) before spending a screening on it. the env check failing on a dumb bug is a waste of points.

## stuff that reverts / goes stale (the footguns, collected)

- **re-uploading the env zip reverts `[agent] user` to root and drops `[verifier]`** - re-add them every time.
- **any change re-stales the whole gate** - prompt edits, env re-uploads, task.toml edits. expect to re-run.
- the **AI-like % badge is cached** - re-run setup to refresh it.
- the **UI is fiddly.** if you drive it with mouse automation it *will* occasionally click the wrong category/button and you'll re-run a bunch of states. happens to the best of us. slow down on the clicks.

{{< figure src="p10.png" alt="the fiddly UI will misclick on you" >}}

## common errors and what they actually mean

- **"cannot create a sandbox" / a Modal error** → Shipd runs the sandboxes on Modal (the AI infra company). this is almost always a transient Modal capacity blip on their side, not your challenge. wait a bit and retry.
- **`[CONVEX M(problems:submitDraft)] ... Server Error`** on submit → backend hiccup. retry; if it persists, it's worth flagging in the discord rather than rebuilding your challenge.
- **a state stuck "Launching" / a stale batch showing old results** → results can be tied to the env that was deployed *when that batch launched*. if you uploaded a new env after launching, you may be staring at a pre-upload result. re-run a fresh batch and verify against the current bundle.
- TODO

## making it realistic (not ctf-y)

the system will literally tell you your challenge "isn't realistic enough." how to get past that:

- **research a real attack** and build around it - a real crypto/web/pwn weakness that exists in the wild, not a contrived puzzle.
- **frame it as a service.** something is running, the agent has to interact with it, probe it, and work the weakness out of how it behaves. a black box it pokes at, not a riddle.
- **no flags, ever.** the win condition is recovering the real thing (a secret, a forged signature, a key) and the verifier checks that, not a magic string.
- **lean on a scenario / story** to make it land - a registry service, a notary, a KMS, a license server. give it a reason to exist.

{{% alert context="info" %}}
steal shamelessly from the official sample (the [badger-merkle-digest](https://shipd.ai/quests/cipher/challenges/s57fdqd582n93qr18fbaktc201881svr?step=overview&example=badger-merkle-digest) example on the cipher challenges page) - it's the reference for tone, structure, and how "realistic" should read lmao.
{{% /alert %}}

## the verifier - getting it right

- `test.sh` is the entry. canonical shape:

{{< prism lang="bash" line-numbers="true" line="4,7" >}}
#!/usr/bin/env bash
set -euo pipefail
mkdir -p /logs/verifier
echo 0 > /logs/verifier/reward.txt      # default to fail
# ... run the tests against the agent's output artifact ...
# on success:
echo 1 > /logs/verifier/reward.txt
{{< /prism >}}

- the reward lives at **`/logs/verifier/reward.txt`** - `1` pass, `0` fail.
- **make it self-contained and opaque.** verify the *real* property (e.g. a signature actually verifies under the library, the recovered bytes match the in-memory secret) - don't just string-match, and don't make the pass condition something the agent can fake. the **gameability** check exists specifically to catch verifiers that can be tricked.
- **clear up the "who runs what" confusion:** during the **environment/oracle check**, the platform runs *your reference solution* to prove reward=1, then the verifier. during the **screening**, the *agent* writes its own solution. your `test.sh` should grade whatever lands in the agreed output path (e.g. `/app/result.txt` or `/output/secret.txt`) regardless of who produced it. you don't pre-place the agent's file - it writes it.

## quick cheatsheet (the gotchas, one screen)

- `[agent] user = "player"`, `[verifier] user = "root"` - re-check after every env re-upload.
- **source + secrets out of the workdir**, in a root-`700` dir. the `/tmp/cipher-baseline` snapshot leaks anything in the workdir regardless of perms.
- pwn: suid binary outside the workdir (workdir perms get stripped), source locked away separately.
- runtime-generated secrets (`os.urandom`) are safe; build-time files are not.
- everything re-stales on change - batch your edits.
- AI-like % is cached - re-run setup to refresh.
- hard-but-solvable: Orion clears it, Nova ≤ ~2/6, 0/6 is fine, *too easy* is the only fatal outcome.
- test your env + reference locally before spending a screening.
- Modal errors = wait and retry, not your fault.
- no flags. realistic. a service to probe, not a riddle.
- sometimes when the infra is down - the agents migth timed-out or api rate-limited, don't panic - just use the re-run option with the indivuial agents to re-run it when the infra is back up.

{{< figure src="p11.png" alt="agents timing out, re-run when the infra is back" >}}

## footnote

if it's your first one: pick a real attack you understand, wrap it in a small service, write the reference exploit *first* (if you can't solve it cleanly, the agents can't either *unless chopped :/*), then build the verifier around the real win condition, then worry about the prompt last. validate locally, then spend the checks.

finally check/ask in the discord - half the stuff in this guide came from people getting stuck out loud and someone going "oh yeah that's the harness bug." you're not the first person to hit any of this.
