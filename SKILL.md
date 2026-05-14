---
name: tiktok-video-auto-publish
description: Publish video materials from a Feishu Base table to TikTok Studio and sync the publish result back to the same table. Use when the user asks to batch or regularly publish newly added Feishu records to a TikTok account, and needs status/link/time fields updated after each publish attempt.
---

# Feishu Tiktok Publisher

## Overview

Execute a closed-loop workflow: pull pending rows from Feishu Base, upload and publish videos in TikTok Studio, then write back publish status, publish link, publish account, notes, and actual publish time.

Default table pattern:
- Source fields: `VIDEO`, `COPYS`, `发布状态`
- Writeback fields: `发布状态`, `TikTok发布链接`, `发布账号`, `发布备注`, `实际发布时间`
- Common status values: `待发布`, `已发布`, `发布失败`, `跳过`

## Workflow

1. Resolve identifiers:
- Convert Feishu wiki URL to `base_token` via `wiki/v2/spaces/get_node`.
- Resolve target table id (`tbl...`) and target view (`待发布（TikTok）` by default).
- Confirm the TikTok account currently logged in.

2. Read pending records from Feishu:
- Query only records in pending view/status.
- Keep fields small: `VIDEO`, `COPYS`, `发布状态`, plus optional schedule fields.
- Skip rows without video attachments; mark as `跳过` or keep pending based on user preference.

3. Publish each record in TikTok Studio:
- Download the attachment locally when needed.
- Open `https://www.tiktok.com/tiktokstudio/upload`.
- Upload video, paste `COPYS` into description, verify visibility and schedule options.
- Publish now unless the row explicitly requests scheduled publish.

4. Collect publish result:
- Detect success signal on TikTok Studio content page.
- Capture canonical TikTok video URL (for example `/@account/video/<id>`).
- Record logged-in account handle used for publish.

5. Write back to Feishu immediately per record:
- On success:
  - `发布状态` = `已发布`
  - `TikTok发布链接` = canonical video URL
  - `发布账号` = TikTok handle
  - `发布备注` = short audit note (for example `TikTok已发布，内容审查中`)
  - `实际发布时间` = current timestamp
- On failure:
  - `发布状态` = `发布失败`
  - `发布备注` = short reason + failing step

## Reliability Rules

- Write back right after each single record, not after the whole batch.
- Never re-publish rows already marked `已发布` unless the user explicitly asks to re-run.
- If publish button is disabled or checks are running, wait and re-check before failing.
- If a row has malformed copy or missing attachment, skip and write a precise note.
- Keep user updated during long batches with short progress messages.

## Minimal Command Pattern

Use `lark-cli` for Feishu read/write whenever possible:
- Read records from table/view.
- Update records with `record-batch-update`.
- Keep payload fields narrow to reduce errors.

Use browser automation for TikTok publish:
- Prefer direct deterministic actions: open page, set description, click publish, verify success marker.

## Output Contract to User

After a run, always report:
- Number of attempted records
- Number published successfully
- Number failed/skipped
- Published links (or at least the first few if large batch)
- Whether Feishu writeback is complete and consistent with view filters

## Safety Notes

- Do not click final publish until video, copy, and account are confirmed.
- If the user asks for manual confirmation before irreversible actions, pause before clicking publish.
- Treat missing permissions or 403 responses as integration issues; fall back to UI flow and document the reason in `发布备注`.
