# Results tracking — Google Sheet setup

The bingo page records **only the final state of the board, and only when a player
reaches Bingo**. Tiles clicked and unclicked along the way are never logged. Each
Bingo appends one anonymous row to a Google Sheet you own:

| Date | Meeting | Bingo Tactics | Other Marked Tactics |
|---|---|---|---|
| 2026-07-10 14:47 | BWC Working Group | Appeal to Emotions; Fake Mirroring; … | Victim Framing; Flooding the Zone… |

- **Bingo Tactics** — the tactics in the winning row/column/diagonal (the FREE space
  is excluded, so a line through the center lists four tactics instead of five).
- **Other Marked Tactics** — every other tactic still flipped when Bingo was reached.

No player data is collected: the request carries only the meeting name (whatever the
player typed in the "What meeting are you observing?" box) and the tactic lists.
Google Apps Script web apps accessed anonymously do not reveal the requester's
identity or IP address to the sheet owner, and the site sets no cookies and stores no IDs.

Games that never reach Bingo are not recorded. Local testing (localhost) is never
recorded — events print to the browser console instead.

## One-time setup (about 5 minutes)

1. Create a new Google Sheet: <https://sheets.new>. Name it e.g. "Disinfo Bingo Results".
2. In the sheet, open **Extensions → Apps Script**.
3. Delete the placeholder code and paste the script below, then save (⌘S).
4. Click **Deploy → New deployment**. Click the gear next to "Select type" and choose
   **Web app**. Set:
   - **Execute as:** Me
   - **Who has access:** Anyone
5. Click **Deploy**, authorize when prompted, and copy the **Web app URL**
   (it looks like `https://script.google.com/macros/s/AKfy.../exec`).
6. In `index.html`, find `const SHEET_ENDPOINT = '';` and paste the URL between the
   quotes. Commit and publish.

New rows appear in the sheet's **Log** tab (check the tab bar at the bottom — the
script creates this tab itself, headers included; don't add column names by hand).

## Updating an existing deployment

If you already deployed an earlier version of the script, paste the new code over the
old, save, then go to **Deploy → Manage deployments → ✏️ Edit → Version: New version
→ Deploy**. Saving alone is not enough — the live URL keeps serving the old code
until you deploy a new version. The URL itself does not change, so `index.html`
needs no edit.

If your **Log** tab still has headers from an older script version (e.g. Date /
Meeting / Type / Tactic), delete or rename that tab; the script will recreate it
with the current headers on the next Bingo.

## The Apps Script code

```javascript
const SHEET_NAME = 'Log';

function doPost(e) {
  const lock = LockService.getScriptLock();
  lock.tryLock(5000);
  try {
    const ss = SpreadsheetApp.getActiveSpreadsheet();
    const sheet = ss.getSheetByName(SHEET_NAME) || ss.insertSheet(SHEET_NAME);
    if (sheet.getLastRow() === 0) {
      sheet.appendRow(['Date', 'Meeting', 'Bingo Tactics', 'Other Marked Tactics']);
    }
    const p = (e && e.parameter) || {};
    sheet.appendRow([
      new Date(),
      String(p.meeting || '').slice(0, 200),
      String(p.bingo || '').slice(0, 1000),
      String(p.marked || '').slice(0, 2000)
    ]);
  } finally {
    lock.releaseLock();
  }
  return ContentService.createTextOutput('ok');
}
```

The `slice` limits cap field lengths so nobody can dump huge payloads into the sheet,
and the lock prevents concurrent plays from clobbering each other's rows.

## Analyzing the data

Each row is one completed game, so "how often was each tactic spotted" needs the
semicolon-separated lists split apart. In a blank tab, this formula counts every
tactic across both columns of the Log:

```
=QUERY(FLATTEN(ARRAYFORMULA(TRIM(SPLIT(TEXTJOIN("; ",TRUE,Log!C2:D),";")))),
       "select Col1, count(Col1) where Col1 <> '' group by Col1 order by count(Col1) desc
        label Col1 'Tactic', count(Col1) 'Times spotted'")
```
