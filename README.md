# Persona

Persona is a local news assistant built with n8n. Users register and authenticate in a chat, choose their interests and delivery preferences, and receive personalized newsletters immediately or on a schedule.

## Main features

- Registration, login, logout and session-based authentication.
- Password hashing and temporary account lockout after repeated failures.
- User settings for topics, email, delivery time, timezone and newsletter status.
- A configurable limit of 1–10 articles per topic, with 3 as the default.
- RSS ingestion from BBC Technology, TechCrunch, The Guardian, Bloomberg and The New York Times.
- Groq summaries and deterministic primary-topic classification.
- Topic-grouped HTML newsletters with unique articles and safe links.
- Immediate delivery with `newsletter` and scheduled delivery with `subscribe`.
- Account deletion and newsletter unsubscribe flows.

## How it works

```text
Chat
  -> registration or login
  -> user settings in persona_users
  -> newsletter request
  -> select articles by topic
       -> prefer the last 48 hours
       -> fill shortages from the last 7 days
       -> run News Fetch once when the pool is still short
  -> build grouped HTML
  -> send through SMTP
```

Each article has one primary topic and can appear only once in a newsletter. If a user selects five topics and requests three articles per topic, Persona requests 15 unique articles. When fewer recent articles are available, it sends the best available selection and reports the actual and requested totals.

## Workflows

| File | Workflow | Purpose |
|---|---|---|
| `workflows/persona-setup.json` | Persona Setup | Creates the user and session Data Tables. |
| `workflows/persona-chat.json` | Persona workflow | Chat, registration, authentication and user commands. |
| `workflows/persona-news-fetch.json` | Persona News Fetch | Reads RSS feeds, deduplicates articles and enriches them with Groq. |
| `workflows/persona-newsletter.json` | Persona Newsletter | Allocates articles, performs one bounded top-up and sends the email. |
| `workflows/persona-newsletter-scheduler.json` | Persona Newsletter Scheduler | Finds subscribed users whose local delivery time is due. |

The exported workflow IDs are not secrets. Data Table IDs, workflow references and credential references may need to be selected again after importing into another n8n instance.

## Start n8n

The project was developed with n8n `2.46.3` in Docker.

```bash
docker start persona-n8n
```

If the container does not exist yet:

```bash
docker run --name persona-n8n \
  -p 5678:5678 \
  -e N8N_SECURE_COOKIE=false \
  -v persona_n8n_data:/home/node/.n8n \
  n8nio/n8n:2.46.3
```

Open [http://localhost:5678](http://localhost:5678). Prefer `localhost` instead of an insecure remote address when running without HTTPS.

## Import order

1. Import `persona-setup.json` and execute it once on a new instance.
2. Confirm that `persona_users` and `persona_sessions` exist.
3. Import `persona-news-fetch.json`, run it once to create `persona_articles`, and reconnect that table if necessary.
4. Import `persona-newsletter.json` and reconnect `persona_users`, `persona_articles`, News Fetch and SMTP.
5. Import `persona-chat.json` and reconnect the user/session tables, Newsletter workflow and password-hashing credential.
6. Import `persona-newsletter-scheduler.json` and reconnect `persona_users` and Newsletter.
7. Run manual tests before publishing Chat, News Fetch or Scheduler.

The JSON exports are intentionally inactive. Publishing News Fetch enables its hourly trigger; publishing Scheduler enables scheduled delivery.

## Credentials

Credentials stay inside n8n. Never put passwords, SMTP secrets or API keys in workflow JSON or Git.

### Groq

Create a Groq API credential and select it in `Groq Chat Model` inside Persona News Fetch. The current workflow uses `openai/gpt-oss-20b`.

- Normal hourly/manual ingestion processes at most 10 pending articles.
- Newsletter top-up processes at most 20 pending articles.
- AI calls are sequential and delayed to reduce free-tier rate-limit errors.
- Invalid or unavailable AI output falls back to deterministic topic classification.

### SMTP

Create an SMTP credential and select it in `Send Newsletter Email` inside Persona Newsletter. Configure the host, port, username, password and SSL/TLS values supplied by the email provider. Set **From Email** to an address authorized by that SMTP account.

## Supported topics

```text
technology
business
science
world
politics
health
sports
environment
culture
```

Topics are entered as a comma-separated list, for example:

```text
technology, business, science, sports, world
```

## Chat commands

```text
help
register
login
settings
topics
articles
schedule
email
newsletter
subscribe
unsubscribe
logout
delete account
cancel
```

`articles` changes the common article limit for every selected topic. It accepts integers from 1 through 10. `newsletter` requires an authenticated session and always sends to the email stored for that user; chat input cannot provide an arbitrary recipient.

## Personalization rules

1. Normalize a missing or invalid article limit to 3.
2. Process the user's topics in their stored order.
3. Select unused matching articles from the last 48 hours, newest first.
4. Fill each topic's remaining quota with unused matching articles up to 7 days old.
5. If any topic is still short, call Persona News Fetch once with a maximum of 20 articles.
6. Read the article table again and send the best available result.
7. Never duplicate an article ID or URL between sections.

A Groq failure, timeout or rate limit does not block delivery when matching articles are already available.

## Manual test

1. Run Persona News Fetch once or confirm that `persona_articles` contains enriched rows.
2. Open the Chat Trigger and register with a dummy development password.
3. Choose topics, an article limit, email, delivery time and timezone.
4. Log in if necessary.
5. Enter `newsletter` once.
6. Confirm the topic headings, actual/requested totals and links in the email.

Do not repeat the command after a successful SMTP result unless a second email is intended.

## Scheduled delivery test

1. Use `schedule` to choose a time a few minutes ahead and a valid IANA timezone such as `Europe/Paris`.
2. Enter `subscribe`.
3. Publish Persona Newsletter Scheduler and keep n8n running.
4. Confirm that one email arrives on the expected local date.
5. Enter `unsubscribe` after the test if scheduled delivery should stop.

`last_newsletter_sent_at` prevents another scheduled newsletter for the same user on the same local date.

## Validation

Run the workflow contract tests and validate every JSON export:

```bash
node --test tests/persona-workflows.test.mjs

for workflow in workflows/*.json; do
  jq empty "$workflow"
done
```

## Security and privacy

- Raw passwords are not stored.
- Authentication state expires and is stored separately from user preferences.
- Registration requires explicit consent before the account is created.
- Scheduled delivery requires an active subscription.
- Newsletter recipients come only from the authenticated user's stored record.
- Password hashes, salts and session data are never sent to Groq or SMTP.
- `delete account` requires explicit confirmation before stored user data is removed.
