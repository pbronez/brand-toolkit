---
name: brand-sync
description: Sync brand-brief.md between project directory and an external vault (Obsidian, plain directory, etc.). Handles conflict detection and resolution.
---

# Brand Sync

Synchronize brand-brief.md between the current project directory and an external vault location.

The vault can be anything: an Obsidian vault, a plain directory, a shared drive. The path is configured per-project via the `vault_path` field in your brand-brief.md.

## Process

### 1. Locate Files

Find brand-brief.md in both locations:
- **Project:** Current working directory or codebase path from the brief
- **Vault:** Read `vault_path` from the brand brief's YAML frontmatter

If `vault_path` is not set, ask the user where they want to sync to. Use the `project` field from the YAML frontmatter to suggest a filename (e.g., `{project}.md`).

### 2. Conflict Detection

Do not decide from `last_updated`. It is a date, so two edits on the same day look identical, and a newer date says nothing about whether the older copy holds edits the newer one lacks. Pulling the "newer" copy can silently erase work.

Instead, compare each copy against what both held at the last sync, recorded in `last_synced.hash`.

**Content hash.** Hash the whole file (frontmatter and body) with these frontmatter keys removed: `last_synced`, `vault_path`, `codebase`, and the Obsidian-only `tags` and `aliases`. Those keys legitimately differ between the copies or change during sync. Normalize line endings to `\n` and trim trailing whitespace, then take the SHA-256 (for example, pipe the normalized text through `shasum -a 256`).

Compute the hash of each copy, then:

| Project hash vs `last_synced.hash` | Vault hash vs `last_synced.hash` | Result |
|---|---|---|
| same | same | **In sync.** No sync needed |
| changed | same | **Only project changed.** Offer to push project → vault |
| same | changed | **Only vault changed.** Offer to pull vault → project |
| changed | changed | **Conflict** (unless both hashes are equal, which means in sync) |

Also treat these as a **conflict**:
- Neither copy has `last_synced` (first sync between two existing files) and their hashes differ
- The two copies' `last_synced.hash` values disagree (one copy was restored or replaced outside brand-sync)

**On conflict:** show the key differences between the two versions and ask the user to keep the project version, keep the vault version, or merge. Never resolve a conflict without asking. Keeping one version discards the other's changes, so name what will be lost before the user confirms.

### 3. Sync Actions

Every action that leaves the copies matching ends by **recording the sync point**:
1. Compute the content hash of the resulting brief
2. Write `last_synced: { date: <today, ISO>, hash: <sha256> }` to **both** copies

If the sync point is skipped, every later sync reports a conflict.

**Push (project → vault):**
1. Read project brand-brief.md
2. Copy to vault path, creating the brands/ directory if needed
3. Update vault_path in the project copy
4. Record the sync point
5. Confirm: "Pushed brand-brief.md to vault at [path]"

**Pull (vault → project):**
1. Read vault brand file
2. Copy to project directory as brand-brief.md
3. Update codebase path in the brief if different
4. Record the sync point
5. Confirm: "Pulled brand-brief.md from vault at [path]"

**Merge (conflict, user chose to merge):**
1. Combine both versions field by field. Where both changed the same field, ask the user which value to keep
2. Append both Decision Log histories, deduplicated by entry
3. Write the merged brief to both locations and set `last_updated` to today
4. Record the sync point
5. Confirm: "Merged project and vault versions of brand-brief.md"

**Create vault entry (vault doesn't exist yet):**
1. Read project brand-brief.md
2. Ask user for vault path if `vault_path` is not set
3. Create vault file at the configured path as `{project}.md`
4. If the vault is an Obsidian vault (detected by `.obsidian/` directory), add Obsidian-compatible frontmatter (tags, aliases)
5. Update vault_path in project copy
6. Record the sync point
7. Confirm: "Created vault entry at [path]"

### 4. Vault Formatting

When creating or updating the vault copy:
- Preserve all brand-brief YAML fields
- Keep the markdown body intact
- **If Obsidian vault detected** (`.obsidian/` directory exists in vault root):
  - Add `tags: [brand, {business_type}]`
  - Add `aliases: [{brand_name}]` if brand_name is set
- **Otherwise:** Plain markdown, no extra frontmatter

### 5. Summary

After sync, display:
```
Brand Sync Complete
  Project: [path]
  Vault:   [path]
  Direction: [push/pull/merge/create/none]
  Stage: [current stage]
  Last updated: [date]
```
