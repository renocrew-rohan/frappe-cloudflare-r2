# R2 app updates U1 and U2 (for Yardi reporting, Stage 1)

Branch `yardi/u1-u2-rollback-and-deferred-upload`, based on `8ef54ea`. Spec: `docs/project-plan.md` §5 "Recommended updates" in the Yardi repo.

## U1: accurate rollback logging

When an upload's database request fails later, the upload is rolled back and the app deletes the R2 object.
Before, a failed delete was ignored and the log still said "Rolled Back, R2 object removed".
A bucket lock makes that delete fail.

- If the delete succeeds, the log is "Rolled Back" (Failed), as before.
- If the delete fails, the log is **"Kept"** with status **Failed**, the error, and a note that the object was kept.
- No setting. No change to keys.

## U2: deferred upload during an R2 outage

- **New setting** `keep_file_on_upload_failure` ("Keep File on Server If Upload Fails"), **off by default**, in the Uploads section of Cloudflare R2 Settings.
- With the setting off, behaviour is unchanged: the upload fails, the email rolls back, and "Upload Failed" is logged.
- With the setting on, a failed upload keeps the File on the server disk and logs **"Upload Deferred"** (status **Deferred**). The email is not rolled back. The key is chosen once, at the first attempt, and stored in `r2_key`. The key generator hook still decides it.
- **Scheduled retry** `frappe_cloudflare_r2.tasks.every_fifteen_minutes` (cron `*/15 * * * *`) calls `controller.retry_deferred_uploads()`. It retries every deferred file on each run, with no limit, and does nothing when R2 is disabled. When an upload succeeds, the File moves to R2 as normal, the deferred row becomes **Success** with the note "Uploaded on retry", and an "Uploaded" row is logged.
- **Age for Yardi**: `frappe_cloudflare_r2.file_log.deferred_uploads(older_than_minutes=0)` returns the files still deferred, oldest first, with `file`, `file_name`, `file_url`, `deferred_at`, `age_minutes` and `log`. Yardi calls it with `older_than_minutes=60` for the T6 alert. The alert itself is not in this app, and it sends no email.
- R2 File Log gets the new event "Upload Deferred" and the status "Deferred".
- Nothing new writes to `raw_data`. Only the normal upload path does.

## Files changed

- `frappe_cloudflare_r2/controller.py`: upload split into the request and `finish_upload`; U1 in the rollback; deferral; `retry_deferred_uploads`.
- `frappe_cloudflare_r2/file_log.py`: `deferred_uploads()`.
- `frappe_cloudflare_r2/tasks.py`, `frappe_cloudflare_r2/hooks.py`: 15-minute cron.
- `frappe_cloudflare_r2/cloudflare_r2/doctype/cloudflare_r2_settings/cloudflare_r2_settings.json`: new setting.
- `frappe_cloudflare_r2/cloudflare_r2/doctype/r2_file_log/r2_file_log.json`: new event and status options.
- `frappe_cloudflare_r2/tests/test_cloudflare_r2.py`: two tests (below).

## Tests

- `test_failed_delete_after_rollback_is_logged_as_kept`: a locked bucket refuses the delete. The log is "Kept"/Failed, the object still exists, and there is no "Rolled Back" row.
- `test_deferred_upload_retries_after_outage_and_reports_age`: a bad bucket defers the upload, and the email is kept. `deferred_uploads` reports the age, and the 60-minute filter works. A retry during the outage leaves the file deferred. After the outage, the retry uploads the file under the stored key, and the log is Success.

Tests run against moto, as the existing suite does. No real Cloudflare or credentials are used.

## Open

- Overlapping cron runs could upload the same file twice. Each run is one file at a time, and the second upload overwrites the same key with the same bytes. A file lock would prevent this. Not added.
- The retry keeps trying forever with no alert from the app, as the spec says. Yardi raises T6.
