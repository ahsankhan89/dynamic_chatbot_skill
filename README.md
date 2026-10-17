# dynamic_chatbot_skill

A Claude skill (`dynamic-chatbot-builder`) that builds a natural-language, read-only chatbot over any database.

Each time you use it, it asks for:

- view or table names
- column names and what each one means
- vocabulary (phrases your users type and what they mean)
- special business rules and ambiguity rules
- theme color
- connection type (TNS is always asked fresh)

It then writes a per-project `chatbot_profile.md`, a `DOMAIN_REFERENCE.md` knowledge file and an optional themed chat page. Vocabulary you add is merged into the profile and kept for later runs.

## Security

The skill never copies or stores database credentials, TNS values or Claude API keys. Generated `.env.example` files contain placeholders only.

## Install

Upload `SKILL.md` as a custom skill in Claude (Settings > Capabilities > Skills), then start a chat and type `/dynamic-chatbot-builder`.
