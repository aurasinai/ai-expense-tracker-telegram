# Setup guide

Setup takes about 15 minutes: prepare the sheet, import the workflow, add three credentials, fill in *Configuration*, then allow your own chat ID.

## Requirements

- An n8n instance (Cloud or self-hosted) with the AI (LangChain) nodes available
- A Telegram bot token from [@BotFather](https://t.me/BotFather)
- A Google Gemini API key from [Google AI Studio](https://aistudio.google.com)
- A Google account for Google Sheets (OAuth2 credential in n8n)
- For the Telegram Trigger, n8n must be reachable over HTTPS (true for n8n Cloud and most hosted setups)

## Steps

1. **Prepare the Google Sheet.**
   1. Create a new Google Sheet and rename the first tab to `Expenses`.
   2. Put these headers in row 1: `ID`, `Date`, `Description`, `Category`, `Amount`. You can also import `sheet-template/expenses-template.csv`.
   3. Copy the spreadsheet ID from the URL: `https://docs.google.com/spreadsheets/d/`**`THIS_PART`**`/edit`.

2. **Import the workflow.** In n8n, go to *Workflows → Create workflow → ⋯ → Import from File* and pick `workflow/ai-expense-tracker-telegram.json`.

3. **Add the Telegram credential.**
   1. Message [@BotFather](https://t.me/BotFather), send `/newbot` and copy the bot token.
   2. In n8n, open *Telegram Trigger*, create a *Telegram API* credential with the token, then select the same credential in *Show typing*, *Reply: private bot*, *Send reply* and *Send error message*.

4. **Add the Google Gemini credential.** Create an API key in Google AI Studio. In n8n, open *Google Gemini Chat Model* and create a *Google Gemini (PaLM) API* credential with it.

5. **Add the Google Sheets credential.** Open *Add expense*, create a *Google Sheets OAuth2* credential and sign in with the Google account that owns the sheet. Select the same credential in *Read expenses* and *Update expense*.

6. **Fill in Configuration.**

   | Field | Default | What it does |
   |---|---|---|
   | `spreadsheetId` | `YOUR_SPREADSHEET_ID` | The ID copied in step 1 |
   | `sheetName` | `Expenses` | Tab that holds the records |
   | `currency` | `IDR` | Currency the bot uses in replies, such as `USD` or `EUR` |
   | `timezone` | `Asia/Jakarta` | IANA timezone used for "today", "this week" and record IDs |
   | `allowedChatIds` | `YOUR_TELEGRAM_CHAT_ID` | Comma-separated chat IDs that may use the bot |

7. **Activate and allow yourself.**
   1. Activate the workflow (*Active* or *Publish*, depending on your n8n version).
   2. Send any message to your bot. Because your ID is not allowed yet, it replies with "Your chat ID is …".
   3. Paste that number into `allowedChatIds`, save, and send a message again.

8. **Try it.** Send `coffee 25k`, then `how much did I spend today?`, then `change the coffee to 28k`. Check that the rows appear and change in the sheet.

## Node by node

1. **Telegram Trigger** receives each new message.
2. **Configuration** (Edit Fields) holds every setting.
3. **Is allowed user?** (If) checks the chat ID against `allowedChatIds`.
4. **Reply: private bot** answers unknown users with their chat ID and stops.
5. **Show typing** shows "typing…" while the agent works.
6. **Expense Agent** (AI Agent) reads the message and decides which tool to call. It uses:
   - **Google Gemini Chat Model**: `gemini-3.5-flash-lite`, temperature 0.3
   - **Chat Memory**: the last 20 messages, one history per chat
   - **Add expense**, **Read expenses**, **Update expense**: Google Sheets tools on the configured sheet
7. **Send reply** sends the agent's answer.
8. **Send error message** sends a short apology if the agent fails.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Bot never answers | Workflow not active, or n8n not reachable over HTTPS | Activate the workflow; check that your n8n URL is public and uses HTTPS |
| Bot always says it is private | Your chat ID is not in `allowedChatIds`, or has a typo | Copy the number from the bot's reply exactly; separate several IDs with commas |
| "Sorry, something went wrong" | The agent failed, often a missing or expired credential | Open the failed execution in n8n and check the red node |
| Rows go to the wrong tab or fail to save | `sheetName` does not match the tab, or headers differ | Use the exact tab name and the five headers from step 1 |
| Google Sheets errors after a few days | The Google OAuth app is in *Testing* mode, so tokens expire after 7 days | Publish the OAuth app to *In production* in Google Cloud, then reconnect |
| Wrong "today" or wrong week in summaries | `timezone` is not your timezone | Use an IANA name such as `Europe/Berlin` or `America/New_York` |
| Amounts saved as text | The Amount column is formatted as text in the sheet | Format column E as *Number* |
