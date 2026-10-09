# tinaja-bot-base specification

The conventions a bot project follows, and what a language implementation must do to
claim conformance. Written so a Ruby or Go bot behaves like the Python one.

**Status: skeleton.** Every section lists what it has to settle and the questions
still blocking it. Nothing here is settled until its question marks are gone.

Vocabulary is normative and lives in [CONTEXT.md](CONTEXT.md) — **Bot**, **Server**,
**Member**, **Command**, **Reply**, **Cog**, **Mention**, **Glossary term**. This
document uses those definitions rather than restating them.

---

## 1. Scope

What a bot project is, and what the framework refuses to do.

Guarantees:

- A bot is configuration plus code. Everything that can be declared, is.
- A `[replies]` entry and a `CONTEXT.md` term need no Python (or Ruby, or Go).
- A Cog cannot crash the bot at startup. A Cog that fails to import is reported, not
  fatal (`cli.py:129`).

Non-goals — things a reader might reasonably expect and will not find:

- Slash commands / interactions. The framework is prefix- and mention-driven only.
- Per-bot permissions modelling. Permissions come from the invite, not from `bot.toml`.
- Anything that requires a privileged intent other than the ones in `[intents]`.
- Multi-server state. A bot's state is per-run; nothing persists between runs.

> **Q1.1** Is the guarantee list complete? Anything else the framework silently does
> that should be written down or removed?
>
> **Q1.2** Should the non-goals stay non-goals as languages are added?

---

## 2. Repository anatomy

The files a bot project contains, and which are required.

| path | required | purpose |
|---|---|---|
| `bot.toml` | yes | name, prefix, intents, Replies, mention behaviour |
| `CONTEXT.md` | yes | the Server's glossary |
| `cogs/` | yes | one file per Cog |
| `.env` | at runtime | `DISCORD_BOT_TOKEN` only |
| `Dockerfile` | to build | see §11 |
| `<manifest>` | yes | `pyproject.toml`, `Gemfile`, `go.mod` — see §9 |

Rules:

- `cogs_dir` and `context_file` in `bot.toml` rename the last two. Both default as above.
- `cogs/` may not exist. A bot of pure Replies and a glossary is a valid bot.
- The token is never in `bot.toml`. It is read from the environment, or from a `.env`
  next to `bot.toml` (`config.py:42`).
- `.env` is loaded without overriding an already-set environment variable
  (`python-dotenv`'s default). An exported variable wins over the file.

> **Q2.1** Should a missing `CONTEXT.md` be an error or a warning? Today it yields an
> empty glossary and `check` only warns.
>
> **Q2.2** Is "token only, from env" the right rule, or may a bot read extra secrets?
> A Cog can read any env var it likes — is that endorsed or merely tolerated?

---

## 3. `bot.toml`

Every key, its type, its default, and what it means. This is the table a language
implementation ports first.

| key | type | default | meaning |
|---|---|---|---|
| `name` | string | `"Bot"` | Label. Not sent to Discord; the bot's Discord name comes from the application. |
| `prefix` | string | `"!"` | What a Member types to start a Command. |
| `cogs_dir` | string | `"cogs"` | Directory of Cogs, relative to `bot.toml`. |
| `context_file` | string | `"CONTEXT.md"` | The glossary, relative to `bot.toml`. |
| `[intents] message_content` | bool | `true` | Read message text. Privileged — must also be enabled in the Developer Portal. |
| `[intents] members` | bool | `false` | Member list events. Privileged. |
| `[intents] presences` | bool | `false` | Presence events. Privileged. |
| `[replies]` | table | `{}` | One plain-text Command per key, value is the Reply. |
| `replies.fallback` | string | *(absent)* | **Reserved.** The answer to an unknown Command. Not itself a Command. |
| `[mentions]` | table | *(absent)* | Presence of this table enables Mention handling. |
| `mentions.as_prefix` | bool | `false` | `@Bot hello` behaves as `!hello`. |
| `mentions.reply` | string | *(absent)* | Sent when the Bot is Mentioned without a Command. |

Notes:

- `[mentions]` is enabled by its **existence**, not by its contents. `[mentions]` with
  no keys still enables Mention handling (`config.py:52`).
- `fallback` is removed from `[replies]` before Commands are built, so it can never be
  invoked as one (`config.py:47`).
- Unknown keys are ignored. A typo in `cogs_dir` produces a bot with no Cogs and no
  complaint. **This should change** — see Q3.2.

> **Q3.1** Strict or permissive parsing? Strict catches typos at `check` time, which is
> the whole value of `check`. My instinct is strict, with unknown-key errors naming the
> key and the closest match.
>
> **Q3.2** Should `check` fail on a `cogs_dir` that exists but is empty, versus one that
> is misspelled? Today both mean "no Cogs" and only the second is really a fault.
>
> **Q3.3** What is the output type of `!glossary` with no arguments, across languages?
> See §7.

---

## 4. Command recognition

What makes a message a Command, and in what order things are considered.

1. The Bot ignores messages its own author sent.
2. With `mentions.as_prefix`, the prefix may be the Bot's mention instead of `prefix`.
3. The text after the prefix is looked up among registered Commands.
4. If found, the Command runs and no Mention fires.
5. If not found and `fallback` is set and the prefix used was the real `prefix`, the
   fallback Reply is sent (`bot.py:31-41`). A Mention-looking invocation is not a failed
   Command and never reaches the fallback.

Reserved names:

- `fallback` in `[replies]` is never a Command (§3).
- Names collide case-insensitively — Discord Command names are lowercased. **Undefined
  today**: what happens if `bot.toml` declares `Ping` and a Cog declares `ping`?

> **Q4.1** What happens on a name collision between a `[replies]` entry and a Cog
> Command? Today the Cog wins by load order (`bot.py:26-31`), which is unspecified
> enough to be a bug. Recommend: `check` reports it as an error.
>
> **Q4.2** Should aliases exist (§12) and if so do they share this namespace?

---

## 5. Replies and substitution

The exact text a Member receives. **This must be byte-identical in every language.**

Algorithm, in order:

1. Replace `{author}` with the Member's mention.
2. Replace `{prefix}` with the prefix.
3. Replace `{command}` with what was typed after the prefix. Empty in a `[replies]`
   Reply; populated in `fallback` and in `mentions.reply`.

It is a literal `replace`, not a format operation (`cogs/replies.py:5-6`). A stray `{`
or `}` in `bot.toml` cannot break a Reply. **A language implementation using `fmt`,
`%` or `str.format` would raise instead — this rule is not cosmetic.**

Unknown placeholders are left in place, untouched.

> **Q5.1** Is the single-pass `replace` order observable? `{command}` substituted before
> `{prefix}` could change output when a Command is literally named `{prefix}`. Currently
> unobservable and unspecified.
>
> **Q5.2** Should a Reply be allowed to reference the Member's *name* rather than their
> mention? `{author}` always produces `<@1234>`, which renders as a ping.

---

## 6. Mentions

A **Mention** is a message that @mentions the Bot but is not a Command.

Fires when all of:

1. The author is not a bot.
2. The Bot is among the mentioned users.
3. `get_context` — or the language equivalent — says this is *not* a valid Command.

The handler receives `(message, text)` where `text` is the message content with the
Bot's mention token removed and whitespace collapsed (`cogs/mentions.py:20`).

Then, if `mentions.reply` is set, that Reply is sent.

Order matters: the Command check happens **before** Mention dispatch, so
`@Bot hello` under `as_prefix` runs `hello` and fires no Mention.

> **Q6.1** The mention regex is `<@!?ID>`. What about `<@&role>` and `<@#channel>` — the
> Bot can be named by a role mention. Today they are not stripped and end up in `text`.
> Should they be?
>
> **Q6.2** Should a Mention be dispatched when the author is the Bot itself? Today no,
> by the bot-author check. Reasonable, but it should be stated rather than incidental.

---

## 7. Glossary

The `CONTEXT.md` grammar. The format is shared; the parser is per-language and must
produce identical terms.

Grammar:

```
# Title

Free prose, the description.

**Term**:
Definition, one or more lines.
_Avoid_: Synonym to use instead.
```

Rules:

- A term opens with `**Name**:` at the start of a line. The definition continues until
  the next term or an `_Avoid_:` line, and its lines are joined with a single space.
- The first `# ` heading is the title. Later headings end the description.
- Lookup is case-insensitive and whitespace-trimmed (`context.py:25`).
- `_Avoid_` is optional.

Command output, **byte-identical in every language**:

| invocation | output |
|---|---|
| `!glossary`, no terms defined | `The glossary is empty.` |
| `!glossary` | `Glossary terms: A, B, C` — names sorted, comma-space separated |
| `!glossary known` | `**Known**: definition` |
| `!glossary known` with `_Avoid_` | as above, then a newline and `_Avoid_: synonym` |
| `!glossary unknown` | `<@1234> "unknown" is not in the glossary` |

Sorting is by term name, case-sensitive, ascending — Python's default `sorted` on `str`.
A Go implementation using a locale-aware collator will disagree.

> **Q7.1** Is case-sensitive ASCII-ish sorting the right call, or should it be
> documented as "whatever the language's default is"? The former is a rule; the latter
> is a bug waiting to happen.
>
> **Q7.2** Does a term's definition get truncated anywhere? A 2000-character Discord
> message limit applies, and nothing checks it today.

---

## 8. Intents and permissions

Two independent gates. Conflating them is the most common source of confusion, so both
belong here.

| gate | what it decides | where |
|---|---|---|
| Privileged intent | what Discord *sends* the bot | Developer Portal, per application |
| Permission | what the bot may *do* in a server | the invite URL, per server |

Framework rules:

- `[intents]` maps to the first gate only. `bot.toml` says nothing about permissions.
- `message_content = true` is required for prefix Commands and for reading history.
- A Missing intent and a Missing permission both produce a bot that connects and never
  answers, with no log line. **This is the framework's worst failure mode.**
- One documented exception: Discord delivers message content even without the intent
  when the message mentions the bot. So `@Bot hello` works with the intent off while
  `!hello` does not — which makes the intent look enabled when it is not.

> **Q8.1** Should the bot log a readiness line naming the intents it asked for and
> whether it received `MESSAGE_CREATE` events? A silent failure here costs an hour
> every time.
>
> **Q8.2** `members` and `presences` are declared but nothing in the framework uses
> them. Keep them for Cogs, or drop them?

---

## 9. The Cog contract

What every language implementation must provide. **This is the section that makes four
templates one framework.**

A Cog is discovered, constructed with no arguments, and registers Commands and
listeners. Beyond that, each language is free.

Python today:

- `cogs/*.py`, sorted, files beginning `_` skipped (`loader.py:16-17`).
- Imported by file path under the synthetic name `tinaja_base_cogs.<stem>`.
- Every `commands.Cog` subclass **defined in that module** is registered. Re-exported
  ones are skipped, so a Cog loads once (`loader.py:22-23`).
- Constructed as `cog()` in `setup_hook`, before the gateway connects (`bot.py:23`).

A language implementation must decide, and document:

| question | Python's answer | portable? |
|---|---|---|
| how is a Cog file found? | glob `cogs/*.py` | yes |
| how is it loaded? | `importlib` by path | yes — Ruby: `require_relative` |
| how is it registered? | class introspection | yes — Ruby: `const_get` |
| **is anything loaded twice?** | no, by `__module__` | **no — Go has no answer** |
| **can a new file be dropped in and just work?** | yes | **no — Go needs an explicit import list** |

The Go row is the leak. In Go, adding a Cog means adding it to `cogs/all.go`. That is
unavoidable — there is no runtime discovery — and the Go README has to say so plainly
rather than let a contributor discover it.

> **Q9.1** Is "drop a file in `cogs/`" part of the contract (and Go is non-conformant),
> or is the contract "register your Cog through whatever mechanism your language
> offers"? The second is honest and portable. The first is friendlier and Go can never
> satisfy it.
>
> **Q9.2** What is the listener contract beyond `on_mention`? Python has arbitrary
> `on_message` listeners. Should that be portable, or is each language free?

---

## 10. Conformance and parity

What must be identical across languages, and what may differ. **This section is the
reason the other eleven exist.**

**Identical — everything a Member sees or an author writes in configuration:**

- `bot.toml` parsing, defaults, and unknown-key behaviour (§3)
- Reply text after substitution, byte for byte (§5)
- `fallback` behaviour and when it does *not* fire (§4)
- Which messages are Commands and which are Mentions (§4, §6)
- The `text` handed to a Mention handler (§6)
- Glossary terms, lookup, sorting, and all five output forms (§7)

**Free — everything a contributor writes in code:**

- Cog discovery and registration (§9)
- Concurrency model. Python is async, discordrb is blocking, Go is goroutines.
- The shape of the context object a Command receives
- Testing fakes. `tinaja_base/testing.py` is Python-shaped and always will be.

The test: if a Member cannot tell which language a bot was written in, conformance
holds. If a contributor has to learn a second language's runtime to add a Cog, it does
not.

> **Q10.1** Shared fixture corpus, or per-language specs? A corpus — one directory of
> `bot.toml`/`CONTEXT.md`/message fixtures with expected output — is the only way the
> identical list above stays true as languages are added. It costs roughly double the
> per-language test work and is the most important artifact in the repo.
>
> **Q10.2** Should `tinaja-bot check` gain a `--spec` mode that validates a project
> against the corpus?

---

## 11. Generated projects

What `tinaja-bot new "Name" --language <lang>` must produce.

Today it copies `tinaja_base/templates/new_bot/` and substitutes `$bot_name`,
`$bot_slug`, `$framework_source` through `string.Template` (`cli.py:73-84`).

Required in every generated project:

- `bot.toml`, `CONTEXT.md`, `cogs/`, `env.sample`, `Dockerfile`
- `.github/workflows/build-and-push.yaml`
- A manifest for the language, and a one-command way to run tests
- `README.md` stating the language's §9 deviations explicitly

The open design question is where the runtime lives.

| | Python today | vendored | gem/package |
|---|---|---|---|
| dependency | git URL + `rev` | none | published artifact |
| fix reaches existing bots | yes, on `rev` bump | no | yes, on version bump |
| builds without network | no | yes | no |
| `check` must verify | source reachable | nothing | artifact exists |

> **Q11.1** Vendored or packaged? Vendoring retires the whole `origin`-remote lookup,
> ssh→https rewriting, local-path rejection and "this repo must be public for Docker"
> class of problem — and makes every language symmetric. Its cost is that a runtime fix
> no longer reaches bots already generated.
>
> **Q11.2** If vendored, how does an existing bot pick up a runtime fix?
>
> **Q11.3** Is `--language` required, or does it default to Python?

---

## 12. Open questions, consolidated

Unanswered design decisions, in the order they block work.

| # | question | blocks |
|---|---|---|
| Q10.1 | shared fixture corpus, or per-language specs | everything |
| Q9.1 | is Cog discovery part of the contract | Go template |
| Q11.1 | vendored runtime or published package | every template |
| Q3.1 | strict or permissive `bot.toml` parsing | `check`, all templates |
| Q4.1 | Reply/Cog name collisions | `check` |
| Q8.1 | log intent readiness at startup | runtime, all languages |
| Q7.1 | glossary sort order | Go, JS |
| Q6.1 | role/channel mention stripping | Mentions, all languages |
| Q5.2 | `{name}` as an alternative to `{author}` | Replies |
| Q12.x | aliases, argument arity, cooldowns, role gating, embed formatting | `bot.toml` schema |
| Q2.1 | missing `CONTEXT.md`: error or warning | `check` |

The `bot.toml` schema questions (Q12.x) are the highest-value ones for the Server's
Members, because they raise the floor: how little someone has to know to ship a
working Command. They are also language-agnostic — the work pays off in all four
templates.