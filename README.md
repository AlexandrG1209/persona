# Persona

Persona is a news assistant built with n8n.

Users can create an account, choose their favorite topics and receive a personalized newsletter by email.

## Features

- Registration and login
- User sessions
- Topic selection
- Email and delivery schedule settings
- 1–10 articles per topic
- Immediate and scheduled newsletters
- Groq summaries and topic classification
- Subscribe, unsubscribe and account deletion

## Workflows

The project contains five n8n workflows:

| File | Purpose |
|---|---|
| `persona-setup.json` | Creates the user and session tables |
| `persona-chat.json` | Handles chat commands, registration and login |
| `persona-news-fetch.json` | Downloads and analyzes news articles |
| `persona-newsletter.json` | Builds and sends newsletters |
| `persona-newsletter-scheduler.json` | Sends scheduled newsletters |

All workflow files are located in the `workflows/` directory.

## Start n8n

Start the existing Docker container:

```bash
docker start persona-n8n
```

Then open:

[http://localhost:5678](http://localhost:5678)

To create a new container:

```bash
docker run --name persona-n8n \
  -p 5678:5678 \
  -e N8N_SECURE_COOKIE=false \
  -v persona_n8n_data:/home/node/.n8n \
  n8nio/n8n:2.46.3
```

## Import workflows

Import the workflows in this order:

1. `persona-setup.json`
2. `persona-news-fetch.json`
3. `persona-newsletter.json`
4. `persona-chat.json`
5. `persona-newsletter-scheduler.json`

Run Persona Setup once to create the user and session tables. Run Persona News Fetch once to create and fill the article table.

After importing, check the Data Table, credential and sub-workflow selections in every node.

## Credentials

Persona needs:

- A Groq API credential in Persona News Fetch
- An SMTP credential in Persona Newsletter
- A crypto credential for password hashing in Persona workflow

Workflow JSON files contain credential references, but they do not contain passwords or API keys.

## Topics

Available topics:

```text
technology, business, science, world, politics,
health, sports, environment, culture
```

The user can choose from 1 to 10 articles per topic. The default value is 3.

Persona first selects articles from the last 48 hours. If there are not enough articles, it can use articles up to 7 days old. The same article is never included twice in one newsletter.

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

Use `newsletter` to send a newsletter immediately. Use `subscribe` to enable scheduled newsletters and `unsubscribe` to disable them.

## News sources

Persona currently uses:

- BBC Technology
- TechCrunch
- The Guardian
- Bloomberg
- The New York Times

## Security

- Passwords are hashed before storage
- Sessions expire automatically
- Registration requires user consent
- Email is sent only to the address stored in the user account
- Secrets remain inside n8n credentials
