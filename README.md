# ai-article-pipeline

An n8n workflow that checks security advisories every morning, has Claude write a draft article, **verifies the draft before it goes anywhere**, saves it to WordPress as a draft, and notifies Discord.

Source: [CISA Cybersecurity Advisories](https://www.cisa.gov/cybersecurity-advisories) (RSS). Output: English or Japanese.

---

## What makes it different

Most "AI blog automation" tools generate text and publish it. This one checks the text first.

- **Verification before publishing.** Every draft goes through a check step. If the AI invents a CVE ID that is not in the source advisory, the draft is **blocked** and the reason is sent to Discord. Numbers that do not appear in the source, missing source links, missing sections, and length problems are flagged as warnings.
- **Copy check (English mode).** Measures how much of the draft repeats the source word for word, so the output is a real summary, not a paraphrased copy.
- **Low cost.** About **$0.0016 per article** (Claude Haiku 5.5, measured on a real run). The cost of every article is logged.
- **Human in the loop.** The workflow only creates **drafts**. Nothing is published without a person reviewing it.
- **No duplicates.** Articles that were already written (per language) are skipped.
- **English / Japanese.** One switch changes the output language, the checks, and the duplicate rules.

## How it works

```
Schedule (every day 9:00) / Manual
        │
        ▼
Settings (lang = en / ja)
        │
        ▼
RSS Read (CISA) ──► Already written? (n8n Data table, link + lang) ──► Limit
                                                                        │
                                                                        ▼
                                                    Claude writes a draft (JSON)
                                                                        │
                                                                        ▼
                                                 Verification (Code node, JavaScript)
                                                      │                    │
                                                    pass                  fail
                                                      │                    │
                                                      ▼                    ▼
                                       WordPress draft (REST API)   Discord: ❌ + reason
                                                      │
                                                      ▼
                                       Log to Data table (model, tokens, cost)
                                                      │
                                                      ▼
                                       Discord: ✅ + edit link + warnings + cost
```

### What the verification checks

| Check | Result if it fails |
|---|---|
| Output is valid JSON with title, summary, body and source URL | Blocked |
| Every CVE ID in the draft exists in the source advisory | Blocked |
| Source URL matches the RSS item and is linked in the body | Blocked |
| Title length | Blocked |
| Draft repeats more than 50% of its 3-word phrases from the source (English) | Blocked (30–50%: warning) |
| Numbers in the draft that cannot be found in the source | Warning |
| Required headings (What happened / Affected products / What to do) | Warning |
| Body length (words for English, characters for Japanese) | Warning |

Example of a blocked draft (test with an invented CVE ID):

```
❌ Verification failed. No draft was created.
Source: CISA Adds Two Known Exploited Vulnerabilities to Catalog
Reason: CVE ID not found in the source: CVE-2026-99999
```

## Tech stack

- **n8n** (self-hosted with Docker Compose)
- **Claude API** (Haiku 5.5 for testing; switchable to Sonnet for higher quality)
- **WordPress REST API** with an Application Password (posts as draft)
- **n8n Data tables** for duplicate checks and cost logging
- **Discord webhook** for notifications
- MariaDB 11 for WordPress

## Setup

Requirements: Docker Desktop, Git, a Claude API key, a Discord webhook.

1. Clone the repository and create a `.env` file (it is listed in `.gitignore`):

   ```
   WP_DB_PASSWORD=your-db-password
   DB_ROOT_PASSWORD=your-root-password
   ```

2. Start everything:

   ```
   docker compose up -d
   ```

   - n8n: http://localhost:5678
   - WordPress: http://localhost:8080

3. In WordPress: set permalinks to "Post name", then create an Application Password for your user.
4. In n8n, create three credentials:
   - `Claude API` (Anthropic API key)
   - `WordPress local` (Basic Auth: WordPress user name + Application Password)
   - `Discord drafts` (Discord webhook URL)
5. Create a Data table `processed_articles` with the columns `Link`, `Title`, `Status`, `Lang`, `Model` (string) and `InputTokens`, `OutputTokens`, `CostUSD` (number).
6. Import `workflows/04-pipeline.json`, connect the credentials, and run it once with **Execute workflow**.
7. To run it every morning, publish (activate) the workflow. It runs while the machine and Docker are on.

No secrets are stored in this repository. API keys and the webhook URL live only in n8n credentials.

## Cost

Measured on a real run (English, Claude Haiku 5.5):

| | Tokens |
|---|---|
| Input | 2,857 |
| Output | 2,637 (of which thinking: 1,599) |
| **Cost** | **≈ $0.0016 per article** |

Every run logs the model, token counts and cost to the Data table, so the real monthly cost can be checked at any time.

## What I learned

- **APIs were the hardest part.** Getting the keys, credentials and authentication right took the most time. At one point I put the Discord webhook URL into the Claude credential and got "Unauthorized". Keeping each secret in its own named credential fixed it and keeps secrets out of the exported workflow.
- **n8n was quick to pick up.** Connecting nodes made sense fast. The real work was in the parts around it: the prompt, the verification code, and handling edge cases.
- **The check step caught real problems.** The first draft failed with "Unterminated string" because the model's thinking used up the token limit and the article was cut off. The verification stopped it instead of publishing a broken post. Raising the limit and asking for a shorter body fixed it.
- **Tested the failure path on purpose.** I asked the model to insert a fake CVE ID to confirm the workflow blocks it.

## Roadmap

- When a draft fails, send the reason back to Claude and let it rewrite the draft automatically
- Google Sheets as an input (topics) and approval board
- Adjustable article length per client
- Publishing to static sites (Astro) through GitHub pull requests

---

## 日本語の概要

CISA のセキュリティ情報を毎朝 RSS から取ってきて、Claude が記事の下書きを書き、**公開前にチェックして**、WordPress に下書きとして保存し、Discord に知らせる n8n のワークフローです。

- 元の記事にない CVE 番号を AI が書いたら止めて、理由を Discord に送ります
- 1記事あたり約 $0.0016（Claude Haiku 5.5 で実測）
- 公開はせず、下書きだけを作ります（人が確認してから公開）
- 日本語と英語をスイッチ1つで切り替えられます
