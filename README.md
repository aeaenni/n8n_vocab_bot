# vocab bot 🐙

A Telegram bot that turns news articles into vocabulary lists and tests you every day with active recall (spaced repetition). Built with n8n, Notion, GNews, Wikipedia and OpenAI.

You send a topic. The bot finds a real article on it, picks 15 useful words at your level, saves them to Notion, then quizzes you at the hours you choose.

## The two workflows

| File | What it does | Starts with |
|---|---|---|
| `vocab_bot_main.json` | Everything you type or tap in Telegram: articles, `/add`, settings, `/review`, quiz buttons | Telegram Trigger |
| `vocab_bot_recall.json` | Sends the quiz cards. Runs on a schedule, **and** works as a sub-workflow called by `/review` | Schedule (every hour) + "When Executed by Another Workflow" |

## Commands

| Command | Example | What happens |
|---|---|---|
| any text, or `/article <topic>` | `/article réchauffement climatique` | Finds an article, saves 15 words grouped by theme |
| `/add <word, word>` | `/add prendre du recul, essor` | Saves up to 5 words or expressions, with definition, translation and example |
| `/lang <source> <translation>` | `/lang fr english` | Language you learn + language of the translations |
| `/level <a1…c2>` | `/level c1` | Words and definitions are tailored to your level |
| `/time <hour> [hour]` | `/time 8 20` | Quiz once or twice a day |
| `/words <1…20>` | `/words 10` | How many words per day (split over your quiz times) |
| `/review` | `/review` | A quiz right now, without waiting for your `/time` |
| `/help` or `/start` | | The help message |

Under each quiz card there are two buttons. **🌠 i know this** doubles the word's review interval (1, 2, 4, 8… days, max 180). **🍄 no clue** resets it to 1 day. Only the first tap counts: the buttons disappear once the message is updated.

---

## How it works

### Main workflow

1. **Telegram Trigger** receives every update: messages and button taps. A bot has only one webhook, so one trigger handles everything.
2. **Config** (Set node) holds the Notion database IDs and defaults.
3. **Normalize The Update** (Code) turns the raw update into simple fields (`chat_id`, `command`, `arg`, `route`).
4. **Usable Message?** (If) sends empty or unknown messages to **Send Help**.
5. **Get Settings → Apply Settings** read this chat's languages and level from Notion (defaults if there is no row yet).
6. **Route The Update** (Switch) has 5 branches:
   - **article**: GNews search → read up to 5 article pages → keep the longest readable one → Wikipedia if no article is readable → the AI picks 15 words → checked in code → save loop.
   - **add**: split the words → the AI enriches each one (definition, translation, example, is it a real word?) → checked in code → save loop.
   - **settings**: `/lang`, `/level`, `/time`, `/words` are validated, then written to the Settings database (update the row, or create it).
   - **button**: the tap is validated, the word's interval is recomputed and saved, and the quiz message is updated.
   - **review**: **Start Review** (Execute Workflow) calls the recall workflow as a sub-workflow for this chat.
7. **Save loop** (shared by article and add): check each word for a duplicate, create the new ones in Notion, build one reply, split it under Telegram's 4096-character limit, send.

### Recall workflow

1. Two ways in:
   - **Every Hour** (Schedule): checks every chat, but only sends to chats whose `/time` hour is now.
   - **Review Requested** (sub-workflow trigger): called by `/review` with one `chat_id`, and sends right away.
2. **Run Context** works out which of the two happened.
3. **Get Words** reads the words from Notion, closest review date first.
4. **Pick Due Words** builds the quiz cards:
   - words due today or earlier come first;
   - **nothing due?** the word with the closest review date is sent early. Its schedule isn't changed unless you tap a button;
   - `/review` with no saved words at all replies "no words to review yet".
5. **One Word At A Time** loops: send the card → if the word was due, **Mark As Sent** moves its review to tomorrow (so the second session of the day doesn't repeat it) → wait 1 second → next.

### Sub-workflow vs separate workflow

A **separate workflow** has its own trigger and starts by itself. A **sub-workflow** is started by another workflow with an Execute Workflow node, which sends it data and waits for it to finish, like calling a function.

The recall workflow is both: its schedule makes it run on its own, and its "When Executed by Another Workflow" trigger lets the main workflow call it for `/review`.

---

## How GNews works

**What it is.** [GNews](https://gnews.io) is a news search API. It indexes articles from tens of thousands of news sites and lets a program search them, like a search engine for news that answers in JSON instead of a web page.

**What the bot sends.** **Search GNews** is an HTTP Request node making one GET request:

```
GET https://gnews.io/api/v4/search
    ?q=réchauffement climatique   ← your topic (quotes and brackets removed first)
    &lang=fr                       ← your /lang source language
    &max=10                        ← up to 10 results
    &sortby=relevance              ← best match first, not newest first
    &from=<29 days ago>&to=<13 hours ago>
    &apikey=<your key>             ← added automatically by the "Query Auth" credential
```

The `from`/`to` window matches the free plan: articles become available after a 12-hour delay, and only the last 30 days can be searched. Asking outside that window returns an error.

**What comes back.** For each article: title, a short description, a truncated preview of the content, the URL, the publication date and the source name. **GNews never gives the full article text** on the free plan, only where to find it.

**How the bot gets the text.**
1. **List Articles** keeps the first 5 results.
2. **Fetch Article Pages** downloads all 5 pages, in parallel, like a browser would.
3. **Extract Paragraphs** pulls the `<p>` paragraphs out of each page's HTML.
4. **Pick Best Article** removes cookie banners and "subscribe" lines, then keeps **the page with the most real text**, not necessarily the top result. A page needs at least 800 characters to count.
5. If none of the 5 pages is readable, **Search Wikipedia** is tried, then **Send No Article** asks for another topic.

**Do you need to pay for Le Monde?** No. GNews doesn't sell access to newspapers; it only points to their pages. The bot then opens the page exactly like a visitor who isn't logged in:
- a free Le Monde article is read normally;
- a subscriber-only article shows only its first paragraphs, so it is usually "too short" and the bot moves on to the next result.

You never pay a newspaper, but you only get what it shows for free.

**Free plan limits.** 100 requests a day, up to 10 articles per request, 12-hour delay, 30 days of history, non-commercial use. Each `/article` uses **one** request; Wikipedia and the page downloads don't count.

---

## Course requirements

| Requirement | Where |
|---|---|
| If / Set node | **Config** (Set); **Usable Message?**, **Already Saved?**, **Real Article?**, **Due Today?** (If) |
| Code node | **Normalize The Update**, **Check Article Vocab**, **Compute Next Review**, **Pick Due Words** |
| Router | **Route The Update** (Switch, 5 outputs) |
| API connection | Notion, GNews, Wikipedia and the article pages via HTTP Request |
| Message trigger | **Telegram Trigger** (messages + button taps) |
| Database | Notion: Vocabulary + Settings |
| AI node | **Extract Themes And Vocab**, **Enrich Word** (LLM Chains, OpenAI) |
| Sub-workflow | **Start Review** (main) → **Review Requested** (recall) |
| Error handling | Error outputs → **Send AI Failed**; GNews → Wikipedia → "try another topic" fallback; input checks (**Read Button**, **Word Belongs To Chat?**); error branch → **Send Error Message** |
| Retries | Retry On Fail on the Notion, GNews, Wikipedia and AI nodes (see below) |

---

## Error handling and retries

**Retries** happen inside the same run: the node tries again by itself, waits, and the run continues if it works. The user only notices a small delay.

| Nodes | Tries | Wait | If it still fails |
|---|---|---|---|
| Get Settings, Find Duplicate, Update Interval | 3 | 2 s | Error output → **Send Error Message** (a tip for the user) |
| Get All Settings, Get Words | 3 | 2 s | The scheduled run stops; nothing is sent this hour (shown in red in **Executions**) |
| Create Word Page, Update Settings, Create Settings, Get Word Page, Mark As Sent | 3 | 2 s | The run continues and reports that item ("could not be saved") |
| Extract Themes And Vocab, Enrich Word | 3 | 3 s | Error output → "ai has failed, please try again!" |
| Search GNews, Search Wikipedia | 2 | 2 s | Falls back to the next source |

On the Notion nodes, "Never Error" is **off** so that a busy server (429) or a server error (5xx) counts as a failure and gets retried. GNews and Wikipedia keep "Never Error" on, because the bot reads their error replies to explain what happened; their retries only cover timeouts and network errors.

**The error branch.** **Get Settings**, **Find Duplicate**, **Update Interval** and **Start Review** have their error output connected to **Send Error Message**. If they still fail after their retries, the user gets a message with a tip based on what they were doing, read from the `route` field of **Normalize The Update**:

| The user was… | Tip |
|---|---|
| asking for an article | try a different or broader topic, or try again in a minute |
| adding words | check the spelling, send fewer words at once |
| changing a setting | check the format with /help |
| tapping a quiz button | the word will come back in a later quiz |
| asking for /review | try /review again in a minute |

**Other safety nets.**
- **AI errors:** **Extract Themes And Vocab** and **Enrich Word** have error outputs to **Send AI Failed**, and the Code nodes after them throw an error when the AI's answer is unusable, which goes to the same message.
- **Fallbacks:** if GNews finds nothing readable, the bot tries Wikipedia, then asks for another topic (**Send No Article**).
- **Input checks:** invalid button data or a word belonging to another chat gets **Answer Invalid Button**; bad settings commands get a usage message.
- **Per-item failures:** a word that can't be saved is reported in the reply ("could not be saved") instead of stopping the whole list.

**Not covered:** a crash in a node with no error output, or a failed scheduled recall run, stops silently. Check each workflow's **Executions** tab: failed runs are marked in red and show which node failed and why.

---

## Setup

### 1. Accounts and keys

| What | Where | Used for |
|---|---|---|
| Telegram bot token | Telegram → @BotFather → `/newbot` | The bot |
| Notion integration token | notion.so/my-integrations → New integration | The two databases |
| GNews API key | gnews.io → sign up (free) | Article search |
| OpenAI API key | platform.openai.com | Word extraction (model `gpt-4.1-mini`) |

### 2. Notion databases

Create two databases. Column names must match exactly, including capitals.

**Vocabulary**

| Column | Type |
|---|---|
| Word | Title |
| Definition | Text |
| Translation | Text |
| Part of Speech | Text |
| Sentence | Text |
| Language | Select |
| Source | Select |
| Chat ID | Text |
| Added On | Date |
| Interval | Number |
| Next Review | Date |
| Theme | Multi-select |
| Article | URL |

**Settings**

| Column | Type |
|---|---|
| Chat ID | Title |
| Source Language | Text |
| Target Language | Text |
| Level | Text |
| Recall Hours | Text |
| Daily Words | Text |

On each database, click **••• → Connections** and add your integration, otherwise the API can't see it.

The database ID is the 32-character part of its URL: `notion.so/<workspace>/`**`3f5d42b79906801985bfed0569b48433`**`?v=…`

### 3. Credentials in n8n

Create them under **Credentials → Add credential**:
- **Telegram API**: the bot token.
- **Notion API**: the integration token.
- **Query Auth** for GNews: name `apikey`, value your GNews key.
- **OpenAI**: your API key.

### 4. Import, recall first

To import, open the workflow (or create a new one), select all nodes on the canvas and delete them, then paste the file's contents onto the canvas. Importing into your **existing** workflows keeps their IDs, so **Start Review** keeps pointing at the right recall workflow.

1. **`vocab_bot_recall.json`**
   - In **Config**, paste your two database IDs.
   - Select the Notion and Telegram credentials in each node that needs one.
   - **Save**. Recall must be saved before main can call it.
2. **`vocab_bot_main.json`**
   - In **Config**, paste the **same** two database IDs.
   - Select the credentials in each node: Telegram, Notion, GNews (Query Auth) and OpenAI on the OpenAI Chat Model node.
   - In **Start Review**, check that **Workflow** shows `vocab bot recall` (re-select it from the list if not), and that **Workflow Inputs** has `chat_id` = `{{ $json.chat_id }}`. Click **Refresh** if the field is missing.
   - **Save**.

### 5. Publish

1. Publish (or activate) the **main** workflow (for the Telegram webhook) and the **recall** workflow (for the schedule, and so `/review` can call it). In newer n8n versions, saving doesn't update the live version: publish again after every change.
2. Check that Telegram points at the production URL: open `https://api.telegram.org/bot<YOUR_TOKEN>/getWebhookInfo`. The `url` should contain `/webhook/`, not `/webhook-test/`.
3. Send `/start` to your bot.

**Test vs production.** Telegram allows one webhook per bot. Clicking *Execute workflow* or *Listen for test event* on the Telegram Trigger switches the bot to the test URL and the published workflow stops receiving messages. After any test, unpublish and publish the main workflow again. To watch production runs, use the **Executions** tab.

**Hosting.** Telegram only reaches a public HTTPS address. On your own computer, n8n needs a tunnel and the `WEBHOOK_URL` setting pointing to it, and the bot only works while the computer is on. n8n Cloud is always on. The hourly schedule uses about 720 executions a month.

---

## Reset for a demo

1. **Notion**: in each database, select all rows (click a row's checkbox, then Ctrl/Cmd+A) and delete them. Keep the columns.
2. **Telegram**: in your chat with the bot, *Clear history* or *Delete chat*, so old quiz buttons don't point to deleted words.
3. Send `/start`. The bot uses the defaults again: French → English, level B2, quiz at 8:00, 10 words a day.

To keep your real words: duplicate both databases, empty the copies, and put the copies' IDs in **Config** in the main and recall workflows for the demo.

## Sharing

- **Share the bot**: send people `t.me/<your_bot>`. The bot is multi-user: each chat has its own settings and words. Their words are stored in your Notion, and they use your OpenAI credit and your 100 GNews requests a day.
- **Share the workflows**: send the two JSON files and this README. Exports contain no secrets, only credential names. Each person creates their own bot, Notion databases and keys, and follows the setup above. Replace your database IDs in both **Config** nodes with placeholders before sending.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| The bot doesn't answer at all | The main workflow isn't published, or Telegram still points at the test URL (see Publish) |
| "ai has failed, please try again!" | OpenAI key, credit, or the model was busy after 3 tries |
| "couldn't find any article on this topic" | GNews found nothing readable in the last 30 days and Wikipedia had no page: try a broader topic |
| No morning quiz | The recall workflow isn't published, no words are due, or `/time` is set to another hour |
| `/review` does nothing | **Start Review** still points at the old workflow ID: re-select `vocab bot recall` |
| The bot goes quiet on one command | A node crashed without an error output: open the main workflow's **Executions** tab and click the red run |
| Notion errors | The database isn't shared with the integration, or a column name or type doesn't match |

## Limitations

- GNews free plan: 100 searches a day for everyone using the bot, articles at least 12 hours old.
- Paywalled articles are usually skipped (only their free part is visible).
- The scheduled recall reads the 100 words with the closest review dates across all users; with many active users, some chats could be skipped in a given hour.
