# Newsletter signup — setup

The "JOIN THE NEWSLETTER" button on `world.html` sends email addresses to a
**Google Sheet** you own, using a small **Google Apps Script** as the receiver.
Nothing else is needed — no server, no third-party service, no cost. The sheet
stays private to your Google account, and submissions are sent over HTTPS.

You only have to do this once. It takes about 5 minutes.

## 1. Create the spreadsheet
1. Go to <https://sheets.google.com> and create a new blank spreadsheet.
2. Name it something like **Zen Gorilla Newsletter**.

## 2. Add the script
1. In that spreadsheet, click **Extensions → Apps Script**.
2. Delete whatever code is there and paste this in:

```javascript
function doPost(e) {
  var lock = LockService.getScriptLock();
  lock.waitLock(30000);
  try {
    var ss = SpreadsheetApp.getActiveSpreadsheet();
    var sheet = ss.getSheetByName('Subscribers') || ss.insertSheet('Subscribers');
    if (sheet.getLastRow() === 0) {
      sheet.appendRow(['Timestamp', 'Email']);
    }

    var data = JSON.parse(e.postData.contents);
    var email = (data.email || '').toString().trim();
    var honeypot = (data.website || '').toString().trim();

    // Honeypot filled in => bot. Silently accept and drop it.
    if (honeypot) {
      return ContentService
        .createTextOutput(JSON.stringify({ result: 'success' }))
        .setMimeType(ContentService.MimeType.JSON);
    }

    // Server-side email validation.
    if (/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      sheet.appendRow([new Date(), email]);
    }

    return ContentService
      .createTextOutput(JSON.stringify({ result: 'success' }))
      .setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    return ContentService
      .createTextOutput(JSON.stringify({ result: 'error', message: err.message }))
      .setMimeType(ContentService.MimeType.JSON);
  } finally {
    lock.releaseLock();
  }
}
```

3. Click the **Save** icon (💾).

## 3. Deploy it as a web app
1. Click **Deploy → New deployment**.
2. Click the gear icon next to "Select type" and choose **Web app**.
3. Set:
   - **Description:** anything (e.g. "Newsletter receiver")
   - **Execute as:** **Me**
   - **Who has access:** **Anyone**
4. Click **Deploy**.
5. Google will ask you to **authorize** — approve it (you may need to click
   "Advanced → Go to (unsafe)"; it's safe because it's your own script).
6. Copy the **Web app URL** it gives you. It looks like:
   `https://script.google.com/macros/s/AKfyc.../exec`

## 4. Connect the website
1. Open `world.html`.
2. Near the bottom, find this line:

   ```js
   const NEWSLETTER_ENDPOINT = "REPLACE_WITH_YOUR_APPS_SCRIPT_URL";
   ```

3. Replace `REPLACE_WITH_YOUR_APPS_SCRIPT_URL` with the Web app URL you copied
   (keep the quotes).
4. Commit and push. Done — new signups will appear as rows in your spreadsheet.

## Spam protection (honeypot + validation)
The form is protected against bots in three layers. You don't have to do
anything to enable this — it's already built into `world.html` and the script
above — but here's how it works so you know what to expect in your spreadsheet.

1. **Honeypot field.** `world.html` includes a hidden text input named
   `website` (class `newsletter-hp`, pushed off-screen). Real visitors never
   see it, but bots that auto-fill every field will fill it in. If it comes
   back filled, the submission is quietly discarded — no row is added, and the
   bot still sees a "success" message so it doesn't retry.
2. **Client-side email check.** Before sending, the page validates the address
   against `^[^\s@]+@[^\s@]+\.[^\s@]+$` and refuses obviously invalid input.
3. **Server-side email check.** The script re-checks the same pattern and the
   honeypot before appending a row, so a bot that skips the page's JavaScript
   and POSTs to the endpoint directly still gets filtered out.

The upshot: only rows with a plausibly valid email and an empty honeypot ever
reach your **Subscribers** sheet.

> **After editing the script**, you must redeploy for the change to take effect
> — see the first note below.

## Notes
- **If you change the script later**, redeploy with **Deploy → Manage
  deployments → (edit) → New version** so the changes go live. The URL stays
  the same.
- **"Anyone" access** only means anyone can *submit* to the endpoint — no one
  can read your spreadsheet. Only your Google account can open the sheet.
- To stop duplicate emails, you can dedupe in the sheet later, or extend the
  script to check existing rows before appending.
