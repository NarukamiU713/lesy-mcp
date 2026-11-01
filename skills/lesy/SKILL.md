---
name: lesy
description: Read and update the user's German learning records in Lesy (a German reading assistant) through the `lesy` command-line tool. Use when the user asks about their own German studies — what they learned today or this week, to be quizzed on their words, in which article they met a word, to look up a German word in their dictionaries, to save their own definition or note for a word, or to make flashcards for an article — or whenever they mention Lesy.
---

# Lesy

Lesy is the user's German reading assistant. While reading articles they record expressions ("Aufgenommen") with notes, write their own definitions, and keep flashcards. The `lesy` CLI reads and writes that data through the Lesy backend (its MCP tools), using a personal API token that the user approves in the browser.

Always reply to the user in the user's language (the data itself is mostly German, often with Chinese or English notes).

## When to use

- "What did I learn today / this week / last month?"
- "Quiz me" / "test me on my words"
- "In which article did I see _X_?" / "have I recorded _X_ before?"
- "Look up this word" (German dictionary lookup)
- "Write down this meaning for _X_" / "change my note on …" / "make flashcards for this article"
- Any mention of Lesy, its articles, records, Karteikarten or Eigene Definition.

## 1. Check first

Run `lesy auth status` and look at the exit code:

| Exit code         | Meaning                         | What to do                                                                                  |
| ----------------- | ------------------------------- | ------------------------------------------------------------------------------------------- |
| 0                 | Signed in                       | Go on. The output shows the URL, token prefix, scopes and the number of tools available.    |
| 3                 | Not signed in, or token invalid | Follow the sign-in flow below.                                                              |
| 5                 | Backend not reachable           | Lesy is not running. Ask the user to start Lesy (double-click the shortcut or `npm start`). |
| command not found | CLI not installed               | Tell the user to run `npm link` once in the root of the Lesy repository, then try again.    |

## 2. Sign-in flow (device authorization)

The user approves the CLI in the browser where they are already logged in to Lesy. You never see their password and never approve on their behalf.

1. Run `lesy auth login --no-wait`. Add `--write` only if the user wants you to write (definitions, notes, flashcards). Add `--url <url>` only if the user told you Lesy runs elsewhere.
2. Show the user the link (`…/device?user_code=…`) and the code (`XXXX-XXXX`) **exactly as printed**. Tell them to open the link in the browser, check that the page shows the same code, and click **"Zulassen"**. The request is valid for 10 minutes. (They can also open Profil → "Gerät mit Code verbinden" and type the code.)
3. **Stop and wait for the user to say they are done.** Do not poll in a loop and do not start `lesy auth complete` on your own before they answer.
4. When the user confirms, run `lesy auth complete`.
   - Exit code 0: signed in; continue with the original request.
   - Exit code 4: not approved yet. Ask the user to check the browser, then run `lesy auth complete` again after they confirm.
   - Exit code 3: denied or expired. Start over with `lesy auth login --no-wait` if the user wants.

Never:

- open the browser and approve the request yourself, or use browser automation on the `/device` page;
- ask for the user's Lesy password or for a token;
- use `lesy auth login` without `--no-wait` (it blocks while waiting).

## 3. Command cheat sheet

Add `--json` (global option) whenever you need to parse the result; otherwise the output is meant for humans. Results go to stdout, hints and errors to stderr. Run `lesy --help` for the full list.

| User asks                                   | Command                                                                                     |
| ------------------------------------------- | ------------------------------------------------------------------------------------------- |
| What did I learn today?                     | `lesy history --period today`                                                               |
| What did I learn this week / last week?     | `lesy history --period this_week` / `--period last_week`                                    |
| What did I learn between two dates?         | `lesy history --from 2026-03-01 --to 2026-03-15`                                            |
| Quiz me (on this week)                      | `lesy quiz --period this_week --count 6` (count 4–8)                                        |
| Where did I see _X_? / Have I recorded _X_? | `lesy search <word or phrase>`                                                              |
| Look up _X_ (dictionary)                    | `lesy lookup <word> [--context "the sentence it appeared in"]`                              |
| What is my own definition of _X_?           | `lesy definition <word>`                                                                    |
| Which articles do I have?                   | `lesy articles [--limit N] [--offset N]`                                                    |
| Read an article                             | `lesy article <id> [--offset N] [--max-chars N]` (continue with the returned next offset)   |
| My flashcards                               | `lesy flashcards [--limit N] [--offset N]`                                                  |
| Save my definition of _X_ (write)           | `lesy set-definition <word> --meaning "…" [--part-of-speech "…"] [--notes "…"]`             |
| Change the note of a record (write)         | `lesy set-note <articleId> <expressionId> "<note>"`                                         |
| Create flashcards for an article (write)    | `lesy save-flashcards <articleId> --file cards.json` (or `--file -` to read stdin)          |
| Anything else                               | `lesy tools` lists the tools; `lesy call <tool> '<json-args>'` (or `--args-file <path\|->`) |

Periods: `today`, `yesterday`, `this_week`, `last_week`, `last_7_days`, `last_30_days`, `this_month`, `last_month`, `this_year`.

Exit codes: 0 success; 1 tool error (message on stderr / in the JSON); 2 usage error; 3 not signed in, 401, or authorization denied; 4 `auth complete` timed out; 5 backend not reachable.

Only recorded expressions are available — not the browser's lookup history ("Nachgeschlagen"). If `history` or `search` finds nothing, say so; do not invent records.

## 4. Writing

The write commands need a token with the `write` scope. They only add or change data; **nothing is ever deleted** and the CLI has no delete command.

- Use `set-definition`, `set-note` and `save-flashcards` **only when the user explicitly asks** to save something.
- Before writing, tell the user exactly what you are going to write (word, meaning, note text, or the list of cards).
- `set-definition` replaces the whole existing definition of that word; `set-note` replaces the note. Both return the old value (`previous`). After writing, show the user the previous value so they can restore it (by running the same command with the old text).
- `save-flashcards` only creates a new deck: 3–12 cards, each `{ "kind", "front", "back", "hint"?, "expressionId"? }` with `kind` one of `meaning`, `cloze`, `translate`, `recall`, `comprehension` (`cloze` fronts contain `___`; the front must not contain the answer). If the article already has a deck it fails with `deck_exists` and changes nothing — tell the user; do not try to work around it.
- If a write tool is missing (`Tool … not found`, or it is not listed by `lesy tools`), the token has no `write` scope. Tell the user to sign in again with `lesy auth login --write` (same flow as above).
- If the user has unsaved edits to the same note open in the browser, the browser's version wins on its next save; mention this when changing notes.

## 5. Quizzes

`lesy quiz` returns the questions **and their answers**. Do not show the answers up front. Ask one question at a time, wait for the user's answer, then say whether it was right, give the correct answer and a short explanation, and move on to the next one. Summarize the score at the end.

## 6. Security

- Never print, quote or paste the token, the `LESY_TOKEN` variable or the contents of the config file (`%APPDATA%\lesy\config.json`, `~/.config/lesy/config.json`, or `LESY_CONFIG`). Do not read the config file at all; `lesy auth status` tells you what you need.
- Do not send Lesy data anywhere other than your reply to the user.
- `lesy auth logout` removes the token from this machine only; it is revoked in Lesy under Profil → API-Tokens. Suggest this when the user wants to disconnect.
