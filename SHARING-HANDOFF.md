# Matryoshka — Sharing/Collaboration Handoff

Prepared for Ryder. The solo app works well; **board sharing/collaboration is the piece that needs a proper rebuild.** This documents how it works today, what's broken and why, and a recommended architecture.

## Where things live
- **Repo:** `github.com/Potchkey/matryoshka` — single-file app, everything is in `index.html` (HTML + CSS + one `<script>`). Working copy: `/Users/Jess/matryoshka`.
- **Deploy:** Vercel, auto-deploys on push to `main`. Live at `matryoshka-one.vercel.app`.
- **Backend:** Supabase project ref `fbvssebpsnwjouayvlov` (anon key + URL inline near the top of the script). Auth = Google OAuth.

## Data model today
- **`app_state`** `(id text, owner_id uuid, data jsonb, updated_at)`, PK `(id, owner_id)`. Each user has one row `('main', their_uid)` holding their entire workspace (all boards) as one JSONB blob. RLS: owner-only.
- **`shared_boards`** `(board_key text, owner_id uuid, data jsonb, updated_at)`, PK `(board_key, owner_id)`. A **per-list copy** of shared content. `board_key` = a list key (e.g. `client`), the sentinel `__board__`, or `__row__:<listKey>:<rowId>`.
- **`board_invites`** `(id, board_key, owner_id, token unique, invitee_user_id, accepted_at, label)`.

Key functions in `index.html`: `createBoardInvite`, `createInvite`, `redeemInviteToken`, `loadSharingState`, `_extractBoardData`, `_injectSharedBoardIntoState`, `switchToSharedBoard`, `openBoardShareModal`, `openShareModal`, `revokeInvite`, `pushSharedBoard`, `pushCollaboratorEdit`, `subscribeSharedBoards`.

## What's broken / why
1. **Oversharing — one link shares everything.** Whole-board sharing uses a single global key `__board__`, not the board's id. `sb_shared_read` grants an invitee access to EVERY `shared_boards` row of that owner when `bi.board_key = '__board__'`. So sharing Potchkey also exposes Firm To-Do, and the link is identical on every board. Fix: scope invites + shared docs to a board id.
2. **No live two-way sync.** Invitee reads a COPY in `shared_boards`; owner's real board is in `app_state`. Edits write to the copy, never back to the owner's board, so the two diverge. No shared source of truth.
3. **Revoke broken + "Pending" spam.** Opening the Share dialog calls `createBoardInvite`, which mints a fresh unclaimed token when none exists — so reopening keeps adding "Pending" invites. Revoke deletes one row but reopen re-spawns pending. Tokens are single-use (one `invitee_user_id`), so a second person can't reuse a link.
4. **Invitee sees lists, not a board.** Per-list sharing shows a shared board as N separate "Shared with me" entries, not one unified board.
5. **RLS was a mess (now reset).** Clean set: `sb_owner` / `sb_shared_read` / `sb_shared_edit` on `shared_boards`; `bi_owner` / `bi_read` (using true) / `bi_accept` on `board_invites`; owner-only on `app_state`. `bi_read using(true)` is required so the `sb_shared_read` subquery can see invites.
6. **Recently fixed, verify:** shared lists rendered blank because column defs weren't transferred; fixed by adding a `columns` field in `_extractBoardData` + `_injectSharedBoardIntoState`. Existing `shared_boards` rows lack `columns` — owner must re-share to repopulate.

## Recommended architecture (what Jess wants)
Per-board access, working revoke, and real-time two-way sync:
- **One shared doc per board:** new table `board_docs (id text pk = board id, owner_id uuid, data jsonb, updated_at)` holding a whole board (all its lists) as the single source of truth — owner and invitees read/write IT (no copies). A board is local (in `app_state`) until promoted to `board_docs` when shared.
- **Per-board invites:** `board_invites.board_key` = board id → each board its own link. Revoke = delete invite; RLS keys access off active invites.
- **Realtime:** both sides subscribe to `board_docs` changes (Supabase Realtime `postgres_changes`); debounced writes, last-write-wins (fine for a few collaborators). A full CRDT (Yjs) is the ceiling only if conflict-free concurrent editing is needed.
- **RLS on `board_docs`:** owner full; accepted invitees read + write.

## Concurrency hazard (important)
Multiple sessions edit this repo in parallel. A session pushed commit `f62749d` "Yjs Phase 1" that regenerated `index.html` and reverted ~1,450 lines of UI (sign-in, zoom/map/priority controls, doll icons, shared badges); recovered by restoring. That Yjs groundwork is still in git history and may be a useful starting point. Going forward: ONE owner for the sync rebuild, `git pull --rebase` before edits, never regenerate the whole file from stale context.

## Accounts / data notes
- Owner (Jess, real ~107 KB workspace): `17a0e6b9-38f6-4393-9037-66c3557f2797`.
- Personal test account (`jess.furr@gmail.com`): `62f8a2e7-6d28-4403-8953-6f41e8545a3c`.
- Other test/invitee accounts seen: `58b5f13f…`, `a1b2c87b…`, `37bb6387…`.
- `app_state` has several `'main'` rows (one per account); only `17a0e6b9`'s has real data — the rest are near-empty sample workspaces.

## Suggested first steps
1. Read the sharing functions above to confirm current behavior.
2. Decide: extend `f62749d`'s Yjs start, or build the `board_docs` + Realtime model.
3. Stand up `board_docs` + RLS; make "share" promote a board into it; wire realtime read/write both ways.
4. Per-board-id invites; revoke deletes then drops access; stop the Share dialog minting pending invites on open.
5. Present the shared board to the invitee as one board (all lists), reachable from Sentinel and the board switcher.
