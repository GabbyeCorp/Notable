# MyTasks App Specification

## 1. Purpose

MyTasks is a personal organization app for Android and PC. It lets a user keep notes, to-dos, and scheduled tasks together in lists, organize them with hierarchical tags, and continue working when offline. Changes synchronize between the user's devices when they reconnect.

## 2. Goals and scope

### Goals

- Provide a consistent experience on Android phones/tablets and PC.
- Make lists the primary way to organize notes and actionable items.
- Support offline reading and editing, with reliable synchronization across devices.
- Support notes with images, including image previews in list views.
- Support nested tags and nested subtasks with clearly enforced depth limits.

### Initial platform assumptions

- Build the application in Dart using Flutter.
- The initial platforms are Android 15 or later and Windows 11 only. Keep platform-specific behavior isolated so additional platforms can be considered later.
- The first release is for one person's account across their devices. Shared lists and collaboration are out of scope.
- Use Firebase Authentication for authentication, Cloud Firestore for synchronized structured data, and Firebase Storage for synchronized image attachments. Keep Firebase-specific code behind a synchronization interface. Validate and authorize every remote write on the Firebase server side, including writes made by offline clients when they reconnect.
- Use Firebase Anonymous Authentication. The app must not provide account creation, account deletion, or account recovery flows, and users do not need an account provisioned outside the app. Firebase project configuration (including the Android `google-services.json` and the Windows app's Firebase options) may contain public project identifiers such as the Project ID, API key, and Storage bucket; these values are not authorization credentials or secrets. Store authentication tokens using platform-appropriate secure storage.
- Because Firebase anonymous identities are device-specific by default, Firebase provides a shared data-space ID that paired devices use to identify the same synchronized data. Provide a secure in-app device-pairing flow initiated from an already-authorized device. The ID alone must not grant access: the flow must securely authorize the new device without exposing credentials or data to an unintended device, using a short-lived single-use code or QR flow. A user who loses access to every paired device cannot recover the anonymous data space in this release.

## 3. Product terminology

- **List:** A user-created container for items. A list may contain notes, to-dos, and scheduled tasks.
- **Item:** A note, to-do, or scheduled task in a list.
- **Subtask:** A child of a to-do or scheduled task. A subtask is itself actionable and may contain subtasks, up to the defined depth limit.
- **Tag:** A user-created label that may have child tags. Tags are available across lists and can be assigned to any item type.

## 4. Functional requirements

### 4.1 Lists

- Users can create, rename, reorder, and delete lists.
- Users can open a list and view its items in a scrollable list view.
- The list view shows the number of currently visible top-level items at the top. When filters are active, the count reflects only top-level items matching those filters. Each task shows a count of its immediate subtasks; nested tasks show their own immediate-subtask counts.
- Each item row shows an icon identifying its type and its tags as small chips. A note row shows its display date, title, content preview, and image thumbnail when present. Omit the title or content preview when that field is empty; do not reserve blank space for it.
- Users can create, edit, move, and delete items in a list.
- For every saved edit to a persisted entity, record a separate history entry for each changed field, including that field's old value, new value, and the date/time of the change. Do not create entries for unchanged fields.
- Users can clone an item as a new independent item, choosing whether to include its subtasks (recursively) and whether to copy its tag assignments. Cloning preserves the item's content and type-specific fields.
- Users can select multiple items in a list and add or remove tags from all selected items in one operation.
- Users can manually reorder items within a list. The chosen order is persisted and synchronized.
- Users can sort list items by title (A–Z or Z–A), creation date, or modified date (`updatedDate`), in ascending or descending order. A custom sort displays the user's manually chosen order; using another sort must not overwrite it, and returning to custom sort restores that order.
- A list can contain any mixture of notes, to-dos, and scheduled tasks.
- Moving an item between lists must preserve its content, tags, subtasks, and attachments.
- The app must provide an understandable empty state for a list with no items.

### 4.2 Notes

- A note has a title (optional), rich-text content, up to three images, zero or more tags, a created timestamp, and a user-editable display date. The display date defaults to the note's created date.
- Users can create, view, edit, and delete notes.
- Note text supports bold, italic, and strikethrough formatting.
- Users can add note images by selecting them from device storage or taking a photo with the device camera when available and authorized.
- A note may have up to three attached images. Each image must be no larger than 10 MB and use JPEG/JPG, PNG, or WebP format. The app must validate image count, file type, and size, and explain when an image is unsupported, exceeds the size limit, or the note already has three images.
- The list view must render image thumbnails for notes that have images. Notes without images must remain compact and must not show an empty image placeholder.
- Selecting a thumbnail opens that image at a larger size. The note remains readable and editable offline.
- Images and thumbnails must be stored locally and queued for synchronization when offline.
- Deleting a note removes its attachment association; attachment data may be physically removed only when it is no longer referenced and its deletion is synchronized.

### 4.3 To-dos and scheduled tasks

- A to-do has a title, optional description, completion state, zero or more subtasks, zero or more tags, and created/updated timestamps.
- A scheduled task has the same core fields as a to-do and may additionally have a start date/time, an end date/time, recurrence rules, and reminder settings.
- A task can be completed or reopened. Its subtasks have independent completion states.
- Completing a to-do or scheduled task records its completion date/time. Reopening and completing it again updates the recorded completion date/time to the latest completion.
- When a to-do or subtask is completed, display it after incomplete tasks at its current sibling level, regardless of the selected sort; the selected sort orders items within the incomplete and completed groups. Completing a parent does not change the order or completion state of its subtasks. When a scheduled-task occurrence is completed, move it after incomplete tasks in that day's scheduled tasks, then remove it from the active day view. The occurrence remains visible in a Completed Tasks view. Completed tasks and subtasks that are displayed use grayish text and italic styling. Reopening an item restores its original manually chosen position and normal text styling; completion-driven display ordering must not overwrite the saved manual order.
- Users can manually reorder subtasks within their parent task. The chosen order is persisted and synchronized.
- Users can sort subtasks within a parent by title (A–Z or Z–A), creation date, or modified date (`updatedDate`), in ascending or descending order. A custom sort displays the user's manually chosen sibling order; using another sort must not overwrite it, and returning to custom sort restores that order.
- A to-do or scheduled task may have subtasks nested to a maximum of **three subtask levels below the top-level task** (four task nodes including the root). The app must prevent creating a subtask that exceeds this limit and must clearly communicate the limit.
- Subtasks inherit neither tags nor schedule values from their parent. Users may assign tags and, for scheduled-task subtasks, dates/times independently.
- When a parent task is completed, its subtasks are not silently changed. The UI may show incomplete subtasks so the parent completion does not hide unfinished work.
- All user-entered dates include a time. Each scheduled task stores an IANA time-zone identifier, defaulted from the device when the task is created. Display and expand its recurrence in that configured time zone, even if the user travels. Recurrences retain the configured wall-clock time across daylight-saving transitions; the corresponding UTC instant may change. If a local time occurs twice when clocks move backward, create an occurrence at each corresponding instant. If a local time does not exist when clocks move forward, mark that occurrence skipped. Store synchronized date/time values in a time-zone-safe representation.
- Implement scheduled-task recurrence using the Dart/Flutter `rrule` package. Supported patterns are daily, weekly (with selectable days of the week), monthly, and yearly. Recurrence includes an end condition (never, at a date/time, or after a number of occurrences). Editing a recurring task applies the edit to the entire series. Completing or reopening a recurring occurrence affects only that occurrence; each completed occurrence appears separately in the Completed Tasks view with its own completion date/time. The Completed Tasks view defaults to `completedDate` ascending (earliest first). A skipped occurrence, including one skipped because its local time does not exist during a daylight-saving transition, is marked skipped, displayed after incomplete occurrences for that day, and styled gray and italic; skipped occurrences do not appear in the Completed Tasks view. Deleting a recurring occurrence must offer a choice to delete that occurrence or the entire series. Persist recurrence rules and per-occurrence completion/deletion/skip state in stable serialized forms so they can be synchronized and exported.
- Users can configure reminders and snoozes for scheduled tasks. A snooze duration is configured in minutes, hours, days, or months. Notifications must be scheduled locally so they can fire without network access, subject to OS permissions and platform limits. Schedule only the immediate next notification for each task, not notifications for years of future occurrences. After a notification fires or its task's schedule/reminder settings change, recalculate and schedule that task's next notification. A task's reminders must not be scheduled more frequently than once every 15 minutes. The app must explain notification permission status and allow reminders to be changed or disabled. Notification delivery and snooze state must synchronize between paired devices per notification event: when a device syncs that an event was already shown or snoozed, other devices must mark that event shown or snoozed and suppress its duplicate; a snooze creates a new notification event.
- Calendar integration is not included in the initial release.

### 4.4 Tags

- Users can create, rename, reorder, and delete tags, and create child tags.
- Tags may be nested without a product-imposed depth limit in the initial release; the UI must remain navigable for deep hierarchies.
- Any note, to-do, or scheduled task can have zero or more tags, including tags at any hierarchy level.
- Assigning a parent tag does not automatically assign its child tags, and assigning a child tag does not automatically assign its parent.
- From a list, users can add a selected tag to or remove a selected tag from multiple selected items at once. Bulk add preserves all other tag assignments, and bulk remove removes only the selected tag from the selected items. This works across notes, to-dos, and scheduled tasks.
- Users can filter items by one or more selected tags. The filter must make it clear whether selecting a parent includes items tagged only with descendants; initial behavior is exact-tag matching, with an option to include descendants.
- Deleting a tag removes that tag assignment from items but does not delete the items. Deleting a tag with children requires the user to confirm whether to delete the entire subtree or first move/reparent the children.

### 4.5 Search and filtering

- Users can search item titles and text/description content across lists.
- Users can filter a list by item type, completion state, and tags. Tag filters support both inclusion (show items with selected tags) and exclusion (hide items with selected tags), including combining included and excluded tags. Multiple included tags use AND semantics: an item must have every included tag. Multiple excluded tags also use AND semantics for the exclusion predicate: an item is hidden if it has any excluded tag. For example, including A and C and excluding B means an item must have A and C and must not have B. Exact-tag matching is the default; when descendant matching is enabled, a selected tag matches its descendants for both inclusion and exclusion.
- Search indexes the plain text extracted from rich-text note content and searches item titles and text/description content across lists. Search and filtering must work against locally available data while offline.
- Search results must identify the item's list and type.

### 4.6 Offline use and synchronization

- The app must be offline-first: users can view and create/edit/delete lists, items, subtasks, tags, and image attachments without a network connection.
- All committed local changes must be persisted on the device before the UI reports success.
- Local changes must enter a durable synchronization queue and sync automatically when connectivity returns.
- Synchronization must cover list and item changes, manual ordering, tag hierarchies and assignments, task completion dates, schedules and recurrence rules, reminder settings and per-event notification delivery/snooze state, and attachment data.
- Synchronization must include field-level change history entries.
- The UI must indicate offline status and distinguish pending synchronization from synchronization failure. Failures must be retryable and must not discard local changes.
- Deletions must use a synchronized `deleteDate` (tombstone) or equivalent mechanism so deleted data is not restored by another device. Retain minimal `deleteDate` metadata for the lifetime of the data space after deleted content is purged so a stale update from a long-offline device cannot resurrect the entity.
- Synchronization must be idempotent: retrying an operation must not create duplicate lists, items, tags, or attachments.
- For edits made concurrently on multiple devices, preserve both versions when an automatic merge is unsafe. Present a conflict-resolution choice to the user rather than silently losing content. Non-overlapping field changes should be merged where safe.
- Initial release synchronization is between devices signed in to the same Firebase Anonymous Authentication data space using the secure in-app device-pairing flow.
- Deleted content is recoverable for 30 days after its `deleteDate`, then must be permanently removed from synchronized storage. Do not retain deleted content beyond this period. Minimal tombstone metadata, including `deleteDate`, is retained for the lifetime of the data space after content purge to prevent resurrection by long-offline devices.
- Each account has a 2 GB quota for synchronized app data, including database records, change history, image attachments, thumbnails, and deleted data during its 30-day recovery period. For this quota, 1 GB equals 1,024 MB. Warn the user when usage reaches 1 GB and clearly prevent or explain writes that would exceed the 2 GB quota. Device-local backup files do not count toward this cloud quota.
- Users can manually create a backup/export and import a previously created backup. Before exporting, the app asks the user to select a backup destination folder using the platform's folder picker. The backup includes lists, items, rich-text content, note display dates and images, tag hierarchy and assignments, task/subtask structure and ordering, completion dates, schedules, recurrence rules, per-occurrence state, and reminder settings. The backup format must be versioned and validated during import; import errors must be reported without partially overwriting existing user data. The user selects a backup file when importing. Restoring a backup replaces the current data on all paired devices; require explicit confirmation that the current data will be replaced and no longer accessible before proceeding. A restore must be versioned so a device reconnecting with pre-restore local changes cannot merge them into the restored data.
- Backups are not encrypted in this release. Inform users that backup files are unencrypted and are their responsibility to store securely.

## 5. Core data model

All persisted entities have stable globally unique IDs, created/updated timestamps, and the synchronization metadata needed to identify pending changes and deletions.

- **List:** `id`, `name`, `sortOrder`, timestamps, `deleteDate`.
- **Item:** `id`, `listId`, `type` (`note`, `todo`, or `scheduledTask`), `title`, type-specific content, `sortOrder`, timestamps, `deleteDate`.
- **Note content:** rich-text body, optional user-editable display date, and up to three attachment references.
- **Task content:** description, completion state and completion date/time, optional schedule start/end and IANA time-zone identifier, recurrence rule, and reminder settings.
- **Recurring task occurrence:** series ID, stable occurrence ID (including a disambiguator for the two instants when a local time occurs twice), completion state and `completedDate`, deletion/skip state, and notification-event state. Completing or reopening an occurrence changes only this occurrence; editing the task changes the whole series.
- **Notification event:** stable event ID, task/occurrence ID, scheduled date/time, delivery state, and snooze state. A snooze creates a new event linked to the snoozed event; syncing a shown or snoozed event suppresses that event on other devices.
- **Subtask:** task identity, parent-task reference, and `sortOrder`. Parent links must not form cycles, and nesting must not exceed three subtask levels below the root.
- **Tag:** `id`, optional `parentTagId`, `name`, `sortOrder`, timestamps, `deleteDate`. Parent references must not form cycles.
- **Item-tag assignment:** item ID and tag ID, represented separately to support multiple tags per item.
- **Change history entry:** entity ID and type, changed field name, old value, new value, and `updatedDate`. Each entry records exactly one field change.
- **Attachment:** `id`, owning note ID, local file reference, content type (`image/jpeg`, `image/png`, or `image/webp`; JPEG and JPG extensions map to `image/jpeg`), file size (maximum 10 MB per image), image dimensions, thumbnail reference, content hash, upload/sync state, `deleteDate`. A note may reference at most three attachments. Account storage usage counts attachment data and thumbnails against the 2 GB quota.

The local database is the source of truth for the on-device user experience. Use a transactional local store and keep persistence and sync logic out of Flutter presentation widgets. Keep reusable app services and data models decoupled from screen layout so future Android and Windows widgets can surface selected lists or tasks without changing core data behavior. Select and evaluate a Flutter-compatible local database package before implementation; Firebase is the selected remote authentication and synchronization provider.

## 6. User experience and accessibility

- Provide navigation to lists, search, and tag browsing on both form factors.
- Use responsive layouts: compact navigation and touch-sized controls on Android; take advantage of PC screen space without changing data behavior.
- Support keyboard navigation and common PC shortcuts where practical.
- Provide accessible labels and adequate contrast; do not rely on color alone to communicate task state or sync status.
- Confirm destructive actions such as deleting lists, items, or tag subtrees. Provide an undo opportunity for local deletions where practical.
- Show loading, empty, offline, pending-sync, and sync-error states explicitly.
- Show the filtered item count at the top of a list and update it whenever items or filters change.

## 7. Non-functional requirements

- **Data integrity:** Local writes are transactional. An app restart or temporary network failure must not lose a successfully saved local change.
- **Performance:** Opening a typical list and saving a local edit should not require a network request. Image thumbnails should be sized for list display rather than repeatedly decoding full-resolution images.
- **Privacy:** Account data and attachments must be transmitted over secure connections. Firebase authentication tokens and other secrets must use platform-appropriate secure storage. Firebase project configuration values are not secrets and must not be treated as authorization. Backup files are unencrypted in this release; warn users before export.
- **Compatibility:** Support Android 15 or later and Windows 11. Use a current stable Flutter release compatible with those platforms.
- **Maintainability:** Keep platform-specific behavior isolated and define a testable boundary between local persistence and remote synchronization.

## 8. Acceptance criteria

1. A user can create two lists and place notes, to-dos, and scheduled tasks in either list; the count at the top includes only visible top-level items and reflects active filters. Each task's subtask count includes only its immediate subtasks.
2. A note supports bold, italic, and strikethrough text, has a display date defaulting to its creation date, and can contain up to three JPEG/JPG, PNG, or WebP images of at most 10 MB each, selected from device storage or captured with an authorized available camera. Unsupported formats, oversized images, and attempts to add more than three images are rejected with an explanation. Images show as list thumbnails and remain available after restarting offline.
3. A task can contain three nested subtask levels; attempting to add a fourth nested level is blocked with an explanation. Items and subtasks retain their manually selected order after restart and synchronization. Completing a to-do or subtask places it after incomplete siblings regardless of the current sort and styles it grayish and italic; reopening it restores its original position and normal styling. Completing a scheduled-task occurrence styles it grayish and italic, places it after incomplete tasks for that day, then removes it from the active day view. It appears separately in the Completed Tasks view; completing another recurrence creates a separate completed occurrence with its own completion date/time.
4. Tags can be nested, assigned to each item type, and used for inclusion and exclusion filtering according to the documented exact-match/descendant-filter behavior.
5. Completing a to-do records its completion date/time, and completing it again after reopening updates that value. Reopening restores the item's original manual position and normal styling. Completing a scheduled-task occurrence records completion for that occurrence only; reopening restores that occurrence to the active schedule.
6. A scheduled task can recur daily, weekly on selected days, monthly, or yearly using `rrule`; editing applies to the entire series. When a local time occurs twice during a daylight-saving transition, both occurrences are created; when it does not occur, that occurrence is marked skipped, displayed after incomplete occurrences for the day, and styled gray and italic. Reminders can be configured and snoozed in minutes, hours, days, or months. Only the immediate next notification per task is scheduled and recalculated after it fires or the task changes, notification spacing for a task is at least 15 minutes, and an authorized device can deliver it while offline. A shown or snoozed notification event is synchronized and suppressed on other paired devices; snoozing creates a new event.
7. A user can create and edit data in airplane mode, close and reopen the app, and see the changes preserved.
8. A user can securely pair another device from an already-authorized device using a short-lived, single-use pairing code or QR flow; expired or reused pairing credentials are rejected. After connectivity returns, offline changes and image attachments synchronize between paired devices without duplicates.
9. A synchronized deletion remains deleted on all devices, including when a device reconnects after being offline; deleted content is recoverable for 30 days from its `deleteDate` and permanently removed after that. Minimal tombstone metadata remains to prevent resurrection.
10. A user can manually export a versioned, unencrypted backup to a folder they select and import it successfully, including note images; the app warns before export, and invalid backups are rejected with an error and do not partially overwrite existing data. Restoring a backup replaces current data on all paired devices only after the user confirms that current data will no longer be accessible.
11. When two devices make conflicting edits to the same content while offline, neither version is silently discarded; the user can resolve the conflict.
12. The app authenticates users anonymously through Firebase and provides no account creation, account deletion, or account recovery flow. Paired devices access the same user's synchronized data; the app clearly explains that losing all paired devices means the data space cannot be recovered.
13. The app builds and runs on Android 15 or later and Windows 11 from the Flutter/Dart codebase.
14. The app warns when account storage reaches 1 GB and enforces the 2 GB account-wide storage quota, using 1 GB = 1,024 MB.
15. From a list, a user can select multiple notes, to-dos, and scheduled tasks, add a tag to all selected items without changing their other tags, and remove a selected tag from all selected items without changing their other tags.
16. A user can clone an item with or without its subtasks and independently with or without its tag assignments; the clone is a new item and does not alter the original.
17. Editing an entity records each changed field as a separate history entry with its old value, new value, and update date/time; unchanged fields do not get history entries.
18. List items and each subtask sibling group can be sorted by title A–Z or Z–A, creation date, or modified date, in either direction. Completing an item places it after incomplete siblings regardless of the selected sort. Switching from custom order to another sort and back restores the original manually chosen order without changes.
19. The Completed Tasks view shows completed tasks and each completed scheduled-task occurrence separately, ordered by `completedDate` ascending by default. Skipped occurrences are marked skipped and appear after incomplete occurrences for their scheduled day, but are not included in the Completed Tasks view.
20. Every remote write is validated and authorized server-side. A write rejected after syncing from offline remains visible locally, is reported as a sync error, and can be retried or corrected without losing local data.

## 9. Storage quota measurement and upload failures

- The application tracks total synchronized storage usage, including database records, change history, attachments, thumbnails, and deleted data during its 30-day recovery period, using provider metadata and client-side size calculations. Count 1 GB as 1,024 MB.
- Before uploading a file, calculate its size and check whether it would exceed the 2 GB account-wide quota. Validate every write on the Firebase server side and enforce the quota authoritatively so concurrent writes from paired devices cannot exceed it; use provider metadata to reconcile usage as uploads and deletions complete.
- When usage reaches 1 GB, warn the user. If an upload would exceed the quota or the provider rejects it, show a clear error, retain the local file and queued change, and allow retry after space becomes available.
