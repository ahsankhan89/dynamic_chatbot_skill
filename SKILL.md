---
name: "dynamic-chatbot-builder"
description: "Use to start or update a natural-language chatbot over any database: asks view/table names, columns and meanings, vocabulary, rules and theme color, then builds the project profile, knowledge file and themed chat page."
---

# DB Chatbot Builder

Generic engine for building a "ask questions in plain language, get answers from my database" chatbot. This skill contains NO project-specific facts (no people, groups, tickets, GM-IT, etc.). Everything specific lives in a per-project **profile file** (`chatbot_profile.md`) kept in that chatbot's own project folder.

## Hard rules (never break)
1. NEVER copy, reuse or store a database connection string, TNS/DSN, username, password or Claude/Anthropic API key from any earlier project or conversation. Always ASK for the TNS fresh for a new project. In generated files use placeholders only (`DB_USER=`, `DB_PASSWORD=`, `DB_DSN=<your TNS here>`, `CLAUDE_API_KEY=<your key here>`). Real values are filled in by the user on their own machine.
2. NEVER run SQL or touch the user's database. Only provide queries/scripts; the user runs them and pastes results.
3. Chatbot is READ-ONLY: generated guidance must allow a single SELECT only, on the listed views/tables only, with a row cap and timeout, and recommend a DB-level read-only role.
4. Do not carry vocabulary or rules from one project into another. Each project has its own profile. Only copy from another profile if the user explicitly asks.
5. Match the user's language register (English / Roman Urdu / Hinglish). Keep questions short, one topic at a time, using simple words.
6. Do not invent column meanings or business rules. If unknown, ask.

## Step 0 - New or existing?
Ask: "New project, or existing? If existing, paste or point me to its `chatbot_profile.md`."
- Existing: read the profile, show a short summary (views, column count, vocabulary count, rules count, theme), then ask what to add/change. **Merge, never overwrite**: keep all existing entries unless the user explicitly says remove/replace. Show a diff of what was added before saving.
- New: run the interview below.

## Step 1 - Interview (ask in this order, wait for answers)
1. **Project name** and one-line purpose (what questions will users ask?).
2. **View or table names** - ALWAYS ask, never assume. Schema-qualified (e.g. `SCHEMA.VIEW_NAME`). One or many.
3. **Columns - for each view/table**: column name + what it means + how it is used. Example format: `IS_PENDING = Y/N flag, used to find pending tickets`. Ask for: ID columns, name/person columns, date columns (and which date to use for which question), status/flag columns and their allowed values, numeric measures, and any column that is unreliable or often NULL.
4. **Which view answers which kind of question**; if several views exist, whether they may be joined (default: avoid joins, prefer one view). Note one-row-per-X vs many-rows-per-X and the counting rule (e.g. COUNT(DISTINCT id)).
5. **Vocabulary / synonyms**: words and phrases users actually type and what each means (term -> meaning/column/value). Ask for several examples. User may add more any time.
6. **Special business rules**: "when the user says X, always do Y" (e.g. definition of pending/active, exclusions, default date field, defaults when no time is given, how to read 'recent').
7. **Ambiguity rules**: names/terms that could mean 2+ things, and what the bot must do (ask 'did you mean A or B?', list all matches, or auto-resolve). Include the full-name-given case.
8. **Real example questions** with the expected answer style (these become the test list).
9. **Output rules**: count-first sentence? tables? page size? max rows? what to say for out-of-scope questions?
10. **Theme color**: primary color (hex or name), light/dark preference, optional secondary/accent, bot title text, language of UI.
11. **Connection**: ask for the Oracle TNS/DSN alias and DB type. Record only the *kind* (e.g. 'Oracle, TNS alias to be set in .env'). Do not write real credentials anywhere.
12. **What to generate**: knowledge file only / knowledge file + themed chat page (default) / plus full app code.

If the user says "skip" or "none" for a section, record it as empty and move on. Offer to resume later; unanswered items go under `## Open questions` in the profile.

## Step 2 - Write `chatbot_profile.md`
Use exactly this structure:

```
# Chatbot Profile - <project name>
Purpose: <one line>
Last updated: <date>

## Data sources
### <SCHEMA.VIEW_OR_TABLE>
Grain: <one row per ...>
Use for: <question types>
Do not use for: <question types>
Counting rule: <e.g. COUNT(DISTINCT ID)>
#### Columns
| Column | Meaning / how it is used | Notes (values, reliability, date usage) |
|---|---|---|

## Which source answers what
<routing rules, join policy>

## Vocabulary
| User says | Means | Maps to (column/value/group) |
|---|---|---|

## Business rules
1. <rule> (reason / date added)

## Ambiguity rules
- <term>: <what the bot must do>

## Output rules
- ...

## Example questions (test list)
| Question | Expected behavior |
|---|---|

## Theme
primary: <hex> | accent: <hex> | mode: <light/dark> | title: <text>

## Connection (no secrets)
DB type: <Oracle...> | DSN: set in .env (DB_DSN) | Read-only role: <pending/done>

## Open questions
- ...
```

Rules for saved vocabulary: every term the user adds is appended under Vocabulary with the date; existing terms stay. Duplicate term with a different meaning -> ask the user which to keep (or keep both as an ambiguity rule). Never silently drop entries.

## Step 3 - Generate the knowledge file `DOMAIN_REFERENCE.md`
Build it from the profile as the system-prompt reference for the SQL-writing model:
- Role: "Translate the user's question into ONE read-only SELECT on the listed sources; then answer in natural language from real results."
- Include data sources + columns, routing, vocabulary, rules, ambiguity behavior, output rules.
- Include the guardrails: single SELECT only, only listed sources, row cap (default 2000), timeout, no DML/DDL, ask a clarifying question when ambiguity rules apply, reply "I do not know" for non-data questions (greetings allowed).
- Include the response contract the app expects, e.g. JSON `{action: run_sql | ask_clarification | not_a_data_question, sql, message}`.
- Include a short section of the example questions as few-shot guidance.

## Step 4 - Generate the themed chat page (if chosen)
Produce one self-contained `index.html` (inline CSS and JS) that:
- defines theme colors as CSS variables on `:root` from the profile (`--primary`, `--accent`, `--bg`, `--text`), with a dark-mode variant;
- shows the bot title, a message list, an input box and Send button, mobile-friendly;
- POSTs `{message}` to `/api/chat` and renders `{answer, rows?, columns?}`; shows rows as a paginated table (default 15 per page); shows a typing indicator and error message;
- contains NO API key, DSN or credentials.
Also generate `.env.example` with placeholders only (see Hard rule 1).

## Step 5 - Optional full app code
Only if chosen. Python FastAPI + python-oracledb (or the DB type given): `main.py`, `config.py` (reads .env), `db.py` (pool, SELECT-only, row cap, timeout), `sql_guard.py` (single SELECT, allowed sources only, block DML/DDL keywords and multiple statements), `claude_client.py`, `orchestrator.py`, `audit_logger.py` (optional SQL audit log), `requirements.txt`. Credentials come only from `.env`.

## Step 6 - Hand over
Give the user: (1) updated `chatbot_profile.md` (save it in the chatbot project folder - this is the permanent memory), (2) `DOMAIN_REFERENCE.md`, (3) `index.html` and `.env.example` if chosen, (4) a short next-steps list: fill `.env`, ask the DBA for a SELECT-only role on the listed sources, install requirements, run the example questions, report wrong answers.

## Refinement loop
After the user tests, for each wrong answer ask: what should it have answered and why? Turn the lesson into a vocabulary entry, business rule or ambiguity rule in the profile (merge, keep old), regenerate `DOMAIN_REFERENCE.md`, and add the question to the test list.

## Reference example (not part of this skill's logic)
A worked example profile for a ticket-system chatbot is kept in the user's project as `ticket-chatbot-example-profile.md`. Offer it only as a sample of a finished profile; never apply its rules to a new project unless the user asks.
