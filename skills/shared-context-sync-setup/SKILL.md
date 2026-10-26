---
name: "drive-project-sync-setup"
description: "Set up a Claude Project that mirrors a Google Drive folder (plus a local folder linked into it). Guides the manual steps, then creates the daily sync task and runs the first sync."
---

# Drive <> Project Sync Setup

Sets up a Claude Project that mirrors a Google Drive folder, including a local folder linked into it, and keeps the Project current with a daily scheduled task. The person does the steps only they can do, and Claude guides them one at a time and checks each one. Claude does everything else.

## How to run this

- Work through the phases in order. Keep a task list with one item per phase.
- Guide manual steps one at a time. Give click-by-click instructions, wait for the person to say it's done, verify it with tools, and only then move on. Don't assume the person knows Drive or Claude settings well.
- Don't trust "already done" without checking. Where a tool check exists, run it. Where none exists, ask the person to confirm what they see.
- If a check fails, say what you found, give the fix, and check again. Never continue past a failed check.
- Never guess folder IDs or paths. If a search returns several matches, ask the person which one is right. Only use folder paths you have seen in `get_device_info` or `device_list_dir` output.

## Phase 1: Find out where the person is

1. Ask with AskUserQuestion (multiSelect) which of these are already done:
   - Google Drive for Desktop is installed and the local folder is syncing
   - A shortcut to the local folder has been added inside the Drive project folder
   - A Claude Project exists with the Drive project folder attached
   - The Drive project folder is connected to Claude on this computer
2. Ask for the name of the Drive project folder (the one that will hold the shortcut) and the name of the local folder.
3. Check whether this chat is inside a Claude Project (Projects tool, `project_info`). The scheduled task and `claude/context.md` belong to the Project, so Phases 3 to 5 must run from a chat inside it. If this chat isn't in a Project, finish Steps A and B here and then do Step C, which ends by restarting the skill inside the Project.

## Phase 2: Guided manual steps

Skip a step only after its check passes.

### Step A: Install Drive for Desktop and sync the local folder

Tell the person:
1. Download Google Drive for Desktop from Google's site, install it, and sign in with the Google account that owns the Drive project folder.
2. Open Drive for Desktop's preferences (the gear icon in its menu). In the section for folders on this computer, choose "Add folder" (some versions say "Add more folders to sync"), pick the local folder, choose to sync it with Google Drive, and save.
3. Wait for the first upload to finish. The Drive for Desktop icon shows progress.

Check: use the Google Drive connector (`search_files`) to find a folder with the local folder's name. Synced computer folders sit under "Computers" in Drive, usually with a parent named after the computer (for example "My Mac"). Note its folder ID. If it doesn't show up after a few minutes, ask the person to check the Drive for Desktop icon for sync errors.

### Step B: Add a shortcut inside the Drive project folder

Tell the person:
1. Open Google Drive in the browser, go to "Computers" in the left sidebar, open their computer, and find the synced local folder.
2. Right-click it, choose Organize, then "Add shortcut".
3. Pick the Drive project folder and confirm.
4. Know who can see it: anyone with access to the project folder can see the shortcut, but they can only open it if they also have access to the original local folder. If the project folder is shared, decide whether the local folder should be shared too.

Check:
1. Find the project folder's ID with `search_files` (a folder, title equals the name, not trashed). If several match, ask which one.
2. Search for shortcuts in it (`mimeType = 'application/vnd.google-apps.shortcut'`, parent equals the project folder ID).
3. Read the shortcut with `get_file_metadata` and confirm `shortcutDetails.targetId` equals the folder ID from Step A.
4. Keep this as a linked-folder row for the registry: link path, target ID, "shortcut", today's date.

### Step C: Create the Claude Project and attach the Drive folder

Tell the person:
1. In Claude, create a new Project and give it a name.
2. Add the Drive project folder to the Project's files from Google Drive. Google Drive must be connected to their Claude account; if it isn't, connect it first. This brings in the shortcut and the local folder along with the Drive folder.
3. Start a new chat inside that Project, in the Claude desktop app on the computer that has the folders, and run this skill again. It picks up where it left off.

If this chat isn't in a Project, stop here after giving these instructions.

Check, once the chat is inside the Project: call `project_info`, note the Project name, and look for the Drive folder's files in the knowledge list. If they aren't there, ask the person to confirm the folder is attached.

### Step D: Connect the Drive project folder to Claude on this computer

The sync task lists the folder on the computer, so the Drive project folder as it appears on disk must be connected. The scheduled task gets its folder access from here (Phase 4), so this step must pass before creating the task.

Tell the person: in the Claude desktop app, in this chat, open the "+" menu beside the message box, choose "Add folder", and pick the Drive project folder inside their Google Drive location on disk (inside "My Drive"; on a Mac it shows under Google Drive in Finder's sidebar). You can also request it with `device_request_folder_access`. If the device tools aren't loaded, load them first.

If the local folder that the shortcut points to lives outside the Drive project folder on disk (the link shows up as a symlink that `device_list_dir` can't open), offer to connect that local folder as well, the same way. Then the task can read it directly instead of only through the Drive connector.

Check: call `get_device_info`. Note each connected folder's absolute path. The Drive project folder's last path segment must equal the Drive project folder name. List it with `device_list_dir` and confirm the link to the local folder is there (as a symlink or shortcut entry).

## Phase 3: Build the configuration

1. **Root.** From `get_device_info`, take the root local path. `<ROOT>` is its last path segment.
2. **Root Drive ID.** Use the ID from Step B. If you didn't get one, `search_files` for a folder titled `<ROOT>`, not trashed. If several match, pick the one whose children match the local listing. If it is still unclear, ask.
3. **Linked folders.** For every symlink in the local listing and every Drive shortcut anywhere under the root, resolve the target folder ID (`get_file_metadata`, `shortcutDetails.targetId`). Record one row each: link path | Drive folder ID | how resolved (shortcut, registry or name match) | today's date. Never guess an unresolved link; ask the person for the folder ID.
4. Show the person the root and linked folders in a short table and ask them to confirm before writing anything.
5. Write `claude/context.md` with `project_write`. If it already exists, read it first and only add or replace the "Sync registry" section. For a new file, include:
   - Last updated (UTC) with time of day
   - Sync registry: root local path, root Drive folder ID, and the linked-folder table
   - Folder structure, Project docs status, Summary, Open items: "Filled in by the first sync."
   - Update log: "<date>: Setup. Registry written."

## Phase 4: Create the daily scheduled task

The task does nothing useful without access to the folder on the computer and to Google Drive. This phase isn't finished until both are confirmed.

1. Ask what time the sync should run each day (AskUserQuestion): early morning (Recommended), midday, evening, or their own time. Write the cron in the person's time zone with `CRON_TZ=<zone>`, and follow the create_trigger guidance on moving the minute off the hour or half hour.
2. Call `list_triggers` (load it with ToolSearch if needed) and look for an existing sync task for this root. If there is one, ask whether to update it or create a separate one.
3. Before creating it, tell the person: "An approval card will appear listing the folder(s) the task can use. Approve it, because that is what gives the task access to your folder."
4. Call `create_trigger` with:
   - name: `Drive to Project Sync - <ROOT>`
   - prompt: the sync prompt below, verbatim
   - cron_expression: from step 1
   - requires_local_device: true
   - folders: the absolute path of the Drive project folder from `get_device_info`, exactly as shown there, plus the local folder's path if it was connected separately in Step D. Pass these explicitly; don't rely on the app filling them in.
   - notifications: `{"push": true}`
   - initiation: `human_request`
5. **Confirm folder access.** Read the create_trigger result, then `list_triggers`, and check that the task requires this computer and lists the folder path(s). Tell the person what you found in one sentence.
   - If the folders are attached: say so and continue.
   - If they aren't (the approval card was declined or closed, the result shows no folders, or the task doesn't require the computer), give these instructions and wait until the person confirms:
     1. On the computer that has the folder, open the Claude desktop app.
     2. Open the scheduled task "Drive to Project Sync - <ROOT>" and its settings.
     3. Turn on "Require this computer".
     4. Give the task access to the Drive project folder (show the exact path from `get_device_info`) and, if relevant, the local folder. Save.
     If the settings don't offer a way to add the folder, delete the task and create it again from this chat (which has the folder connected), approving the card this time. Then check again with `list_triggers`.
6. **Confirm approvals.** Tell the person in one sentence which approval setting the task got. If its runs will ask for approval, explain that nobody is there to approve a scheduled run, so they should switch the task to "Automatically approve" in its settings if their organization allows it.
7. **Confirm connector access.** Ask the person to open the task's settings and confirm:
   - the Google Drive connector is enabled for the task, plus Google Docs, Sheets and Slides if they're connected. Without Drive access the task can't follow links or read Google files.
   - the task is attached to this Project (it normally inherits the Project it was created from)
   Wait for them to confirm each one.

## Phase 5: First run and verify

1. Run the task once now with `fire_trigger` (load it with ToolSearch). If that isn't available, ask the person to run it manually from the task.
2. Use `send_later` to come back in about 10 minutes, then call `project_info` and check:
   - every Drive file under the root, native and linked, has one `.md` snapshot under `<ROOT>/`
   - each empty folder has a `_folder.md`
   - linked content sits under `<ROOT>/<link name>/`
   - `claude/context.md` has a fresh timestamp, a filled-in folder tree and an update-log line for the run
3. If it failed or nothing changed, check the usual causes in this order and fix before running again:
   - **No folder access:** the run found no connected folder (its log says so, or `context.md` wasn't touched). Go back to Phase 4 step 5 and repeat the folder-access instructions.
   - **No Drive access:** the Drive connector isn't enabled for the task. Repeat Phase 4 step 7.
   - **Computer unavailable:** the computer was asleep or offline, or the Claude desktop app was closed.
   - **Approvals waiting:** the run stopped for an approval nobody answered. Repeat Phase 4 step 6.

## Phase 6: Wrap up

Tell the person, briefly:
- what is set up: the Project, the root folder, the linked folders, the folders the task can access, and the daily run time
- that the computer must be awake, online and running the Claude desktop app when the task runs
- that if they move or rename the Drive project folder on disk, the task loses access, and they should rerun this skill or update the task's folder access
- how to add another linked folder: add a row to the Sync registry in `claude/context.md`, or add another shortcut inside the Drive project folder
- that they can run the task manually any time for an update between daily runs

## Sync prompt (use verbatim as the scheduled task's prompt)

````
# Drive to Project Sync

Keep the attached Project in sync with the Drive folder connected to this task (the "root folder") AND everything linked into it, and keep `claude/context.md` current. No folder names or IDs are written into this prompt: resolve them in section 0 on every run.

## 0. Resolve this project's configuration

- **Root folder.** Call `get_device_info` to get the connected folder's absolute path. `<ROOT>` is the last segment of that path. If several folders are connected, sync each as its own root. If none is connected, stop, log it, and notify.
- **Sync registry.** Read the "Sync registry" section of `claude/context.md` (it may not exist on a first run). It lists the root path, the root Drive folder ID, and every linked folder as: link path | Drive folder ID | how resolved | last verified. Entries the user added by hand take precedence over discovery, but still verify them.
- **Root Drive ID.** If the registry has one, confirm it with `get_file_metadata` (name matches `<ROOT>`, not trashed). Otherwise find it with `search_files` (folder, title = `<ROOT>`, not trashed). If several match, pick the one whose children match the local listing. If it is still ambiguous, log it, notify, and use the local listing only.
- **Root renamed.** If the registry's root name differs from `<ROOT>` but the Drive ID is the same, treat it as a rename: move every snapshot from `<old name>/` to `<ROOT>/` (write new, delete old).
- **Linked folders.** Start from the registry's linked folders, then add any new links found in section 1. For a local symlink that cannot be walked, resolve its Drive target in this order:
  1. A registry entry for that link path.
  2. A Drive shortcut with the same name in the matching Drive parent folder. Read its `shortcutDetails.targetId` with `get_file_metadata`.
  3. A Drive folder with exactly that name, outside `<ROOT>`, not trashed. Accept it only if there is exactly one match.
  If none or several match, never guess. Log it as an unresolved link, notify, and ask the user to add the folder ID to the registry. Keep previously synced snapshots for that link.
- Write everything resolved back to the registry in section 5.

## Scope: what counts as "in the folder"

Pull everything that is either natively inside `<ROOT>` or linked into it. Linked means any of:
- A symlink inside the folder (for example, a local folder symlinked into Drive). Follow it.
- A Google Drive shortcut (`mimeType = application/vnd.google-apps.shortcut`) inside any folder in the tree. Follow it to its target.
- A Drive folder listed in the registry as linked. Always sync it, even when the local symlink cannot be walked.

Linked content is mirrored in the project under the path where it is linked: `<ROOT>/<link name>/...`. Use one consistent link name per target and never create two copies of the same content. Apply the same rules to linked folders as to native ones, recursively, including nested links. Guard against loops by tracking visited folder IDs and paths.

## Ground rules

- Run **all sections (0 to 6)** on every run. Updating only `claude/context.md` is a failed run.
- Drive is the source of truth. Never trust `claude/context.md` (except the registry, which you verify), the session header doc list, or any cached view over a fresh listing.
- Only touch `<ROOT>/` and `claude/context.md` in the project. Linked content goes under `<ROOT>/` as described above.
- If a link cannot be followed (access denied, target missing, unreadable), log it in the update log, notify, and carry on. Do not delete the previously synced snapshots for it unless the target is confirmed deleted.
- Notify (PushNotification with a `<routine_summary>`) only when something meaningful changed or failed. See Report.

## 1. Discover

List what exists on both sides, then diff.

- List the connected folder recursively with `device_list_dir`. Record every folder and file by full path, including empty folders and folders new since the last run. Entries of type `symlink` are links: try `device_list_dir` on the symlink path (recursive). If that is not accessible, resolve the target as in section 0 and list it through the Google Drive connector instead.
- Cross-check with the Google Drive connector (`search_files`) so a stale local listing cannot hide a rename or delete: list folders and files under the root Drive ID by `parentId` recursively, search for shortcuts (`mimeType = 'application/vnd.google-apps.shortcut'`) under the tree, and list every registry linked folder by `parentId` recursively. If the two views disagree, trust the one that shows the newer state and note it in the update log.
- List the project docs with `project_info`. Do not rely on the doc list in the session header or on `claude/context.md`; both can lag.
- Diff the two lists by full project path. Classify each item as added, changed, renamed/moved, deleted, or unchanged.
  - Folders count: a new, renamed, moved, or deleted folder is a change even when it holds no files.
  - Treat the mtime of a Drive file as the signal for "changed".

## 2. Read the content

- Drive `.gdoc`, `.gsheet` and `.gslides` files are pointer stubs. Find each one with the Google Drive connector (`search_files` by title or parentId), then read the real content with Google Docs, Sheets or Slides (or `read_file_content`). For linked folders, files are usually real Drive files already: read them with `read_file_content` by file ID.
- If a file cannot be read, log it in the update log and carry on.

## 3. Sync the project folder tree

This is the main job. Do it before touching `context.md`. The project's `<ROOT>/` tree must mirror Drive (native plus linked content) exactly, because the user sees these folders in the project's file panel.

- **Added file:** `project_write` to `<ROOT>/<subfolder>/<file name>.md` (the extension is replaced by `.md`).
- **Changed file:** overwrite its snapshot.
- **Renamed or moved file:** write the new snapshot, then `project_delete` the old one.
- **Deleted file:** `project_delete` its snapshot.
- **Empty folder (new or existing):** `project_write` to `<ROOT>/<folder>/_folder.md` with a one-line note saying it is an empty Drive folder.
  - A folder that has real files needs no `_folder.md`. Delete a leftover `_folder.md` once the folder has files.
  - Renamed or deleted folders: move or delete everything beneath them, including any `_folder.md`, the same way.
- **Duplicates** (a stub entry plus a text snapshot for the same file): keep the text snapshot, delete the other.
- **Each snapshot holds:** title, Drive path (and link path if linked), UTC timestamp, and the file's actual text (or "title only" / "empty").

### Cache and staleness rules

The project panel can show an old folder tree after writes and deletes. Treat any view as stale until proven fresh.

- Never judge the current state from the file panel, the session header, or a doc list read earlier in the run. The only valid check is a fresh `project_info` call made after the last write or delete.
- Run each `project_delete` and `project_write` once, then re-read with `project_info`. If a deleted path still appears, retry the delete once and re-check. If it still appears, log it as a sync mismatch.
- A folder only disappears from the project when no doc path starts with its prefix. Delete every doc under a removed or renamed folder, including `_folder.md`, or the folder will linger.
- Finish with the last write being `claude/context.md` (see section 5) so the project is modified after the tree changes and the panel is prompted to refresh.
- If a later run still sees a folder that Drive no longer has (or misses one Drive has), assume stale cache first: re-list Drive, re-run `project_info`, and compare path by path before making any change. Do not recreate a folder that exists only in a stale view.

## 4. Verify

- Call `project_info` again (fresh, after all writes and deletes) and compare its doc list to the Drive listing (native plus linked), path by path.
- Every Drive folder (native or linked) must appear as a folder prefix in the project. No project folder may exist that Drive lacks.
- Every Drive file must have exactly one `.md` snapshot. No snapshot may exist without a Drive file.
- Fix any mismatch and re-verify. Do not finish until the lists match.

## 5. Update `claude/context.md`

Read it first, then write the full file back. The last write of the run is this file.

- **Last updated (UTC):** timestamp with time of day.
- **Sync registry:** root local path, root Drive ID, and one row per linked folder (link path | Drive folder ID | how resolved: registry, shortcut, or name match | last verified date). Keep user-added rows. Mark unresolved links "unresolved: add folder ID".
- **Folder structure:** a tree of the root folder, with linked content shown under its link name and marked "(linked)". Mark new folders and files. Mark empty folders.
- **Project docs status:** one line per file, grouped by folder.
- **Summary:** goals, scope, milestones, owners, deadlines, status. Only as stated in the docs, never invented.
- **Open items.**
- **Update log:** one dated line per run covering added, renamed, moved, deleted, updated, snapshots written or deleted, read failures, link-following issues, or "no changes". Keep earlier entries. Compress repeated "no changes" to one line per day.
- The context file must never describe a folder or file the project tree does not contain.

## 6. Report

- Finish with a short summary listing the exact project paths written and deleted in section 3, and the result of the verify check.
- **Notify** (PushNotification with a `<routine_summary>`) only when something meaningful changed: files or folders added, renamed, moved or deleted (native or linked), new doc content, a missed deadline, or a newly resolved link. Also notify when something failed: unreachable folder or connector, a link that cannot be resolved or followed, unreadable file, or a sync mismatch after retry.
- **If nothing changed,** update the timestamp and log line only, with no notification.
````