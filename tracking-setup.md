# Results tracking — Google Sheet setup

The bingo page can log one anonymous row per game event to a Google Sheet you own:

| Date | Meeting | Type | Tactic |
|---|---|---|---|
| 2026-07-10 14:32 | BWC Working Group | spot | Appeal to Emotions |
| 2026-07-10 14:47 | BWC Working Group | bingo | Appeal to Emotions; Fake Mirroring; … |

- **spot** — a tactic was flipped for the first time in a game (one row per tactic).
- **bingo** — a player completed a line; the Tactic column lists the five winning tactics.

No player data is collected: the request carries only the meeting name (whatever the
player typed in the "What meeting are you observing?" box), the tactic, and the event
type. Google Apps Script web apps accessed anonymously do not reveal the requester's
identity or IP address to the sheet owner, and the site sets no cookies and stores no IDs.

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

New rows appear in the sheet's **Log** tab as games are played. Use a pivot table
(Insert → Pivot table) to see tactic counts per meeting or per date.

> Note: if you ever edit the Apps Script code, you must create a **new deployment**
> (or update the existing one via Deploy → Manage deployments) for changes to take
> effect — saving alone is not enough. The URL stays the same when you update an
> existing deployment.

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
      sheet.appendRow(['Date', 'Meeting', 'Type', 'Tactic']);
    }
    const p = (e && e.parameter) || {};
    sheet.appendRow([
      new Date(),
      String(p.meeting || '').slice(0, 200),
      String(p.type || '').slice(0, 20),
      String(p.tactic || '').slice(0, 500)
    ]);
  } finally {
    lock.releaseLock();
  }
  return ContentService.createTextOutput('ok');
}
```

The `slice` limits cap field lengths so nobody can dump huge payloads into the sheet,
and the lock prevents concurrent plays from clobbering each other's rows.

## Related: the public tally counters

Independently of this sheet, the site increments a public anonymous counter per tactic
(powering the "Most-Spotted Tactics" chart on the page). Those counters have no date or
meeting dimension; this sheet is the source for that. See the `TALLY_*` constants in
`index.html`.
