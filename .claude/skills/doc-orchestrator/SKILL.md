---
name: doc-orchestrator
description: AI orchestrator that auto-updates Confluence docs based on meeting notes
allowed-tools: Read, Grep, Glob, Bash, Edit, Write
argument-hint: [path to meeting audio/text file]
---

# Document Orchestrator

An agent that takes meeting notes as input, analyzes which Confluence documents need updates,
and automatically applies changes after user approval.

## Execution Flow

### Step 0: Environment Setup

Before starting, run the following setup:
```
set -a && source .env && set +a && pip install -q openai
```
This loads environment variables from the project's `.env` file and installs dependencies.
If `.env` is missing, inform the user to create one (see README.md for the template).

### Step 1: Gather Context (Optional Pre-input)

Before processing meeting notes, ask the user if they already know which documents are relevant.

AskUserQuestion:
- "Do you have specific Confluence pages related to this meeting?"
  - Yes, I'll provide page URLs → Collect one or more Confluence page URLs from the user
  - No, find them automatically → Skip to Step 2 after meeting input

If the user provides URLs, resolve them to page IDs:

**URL formats to handle:**
- Full URL: `https://nota-dev.atlassian.net/wiki/spaces/SPACE/pages/123456789/Page+Title` → extract page ID `123456789`
- Short URL: `https://nota-dev.atlassian.net/wiki/x/GYHUdw` → use confluence_search or confluence_get_page with the URL to resolve

**For short URLs (`/wiki/x/...`):**
The short URL contains a base64-encoded page ID. To resolve it:
1. Try using confluence_search with the short URL or page title
2. Or call `Bash` to decode: the path after `/x/` is a base64url-encoded 32-bit big-endian page ID
   ```
   echo "GYHUdw" | base64 -d | od -An -tu4 | tr -d ' '
   ```
3. Use the decoded page ID with confluence_get_page

After resolving URLs to page IDs:
- Fetch each page via confluence_get_page
- Store these as **pinned documents** — they will always be included in the analysis regardless of search results

### Step 2: Meeting Notes Input

AskUserQuestion:
- "How would you like to provide meeting notes?"
  - Audio file path → Transcribe via OpenAI GPT-4o audio API (see Audio Transcription section below), then analyze
  - Text/transcript file path → Read the file
  - Paste text directly

After reading the meeting notes, extract:
- Key decisions made
- Action items
- Keywords/topics that may require document updates

### Step 3: Find Related Confluence Documents

Search Confluence for documents related to the extracted keywords.

1. Use confluence_search with keyword-based queries (multiple queries for different decision topics)
2. Merge search results with pinned documents from Step 1 (deduplicate by page ID)
3. Select the most relevant documents from combined results (max 10)
4. Present the document list to the user with verification links

Display each document as:
```
1. [Page Title](https://nota-dev.atlassian.net/wiki/spaces/.../pages/PAGE_ID)
   Owner: {owner} | Last updated: {date} | Space: {space_name}
   Relevance: {brief reason why this page is related to the meeting}
```

AskUserQuestion (multiSelect):
- "The following documents may need updates based on the meeting. Select which ones to review."
- User can click links to verify each page before selecting

### Step 4: Analyze Document Contents

Fetch selected documents via confluence_get_page, then:
1. Summarize current content of each document
2. Compare meeting decisions against document content
3. Identify **specific sections** that need changes (not entire pages)

Classification:
- **Required**: Meeting explicitly decided something that directly conflicts with current document content (e.g., document says "session-based auth" but meeting decided "JWT auth")
- **Recommended**: Document content is not directly contradicted but would benefit from updates for consistency (e.g., an onboarding guide that references the old auth flow)

### Step 5: Propose Changes

For each proposed change, present:
- Target document link + specific section name
- Current content (quoted excerpt)
- Proposed new content (exact text)
- Reason for change (which meeting decision drives this)
- Classification: REQUIRED or RECOMMENDED

AskUserQuestion (multiSelect):
- "Which changes should be applied?"
  - Required changes only (N items)
  - Required + Recommended (N items)
  - Individual selection mode

If individual selection mode is chosen, confirm each change one by one via AskUserQuestion.

### Step 6: Confirm Target Pages

Before applying any updates, show a final confirmation with direct links:

```
The following pages will be modified:
1. [Authentication Policy](https://nota-dev.atlassian.net/wiki/...) — 3 sections changed
2. [API Reference](https://nota-dev.atlassian.net/wiki/...) — 2 sections changed

Please verify these are the correct pages. Proceed?
```

AskUserQuestion:
- "Confirm these pages for update?"
  - Yes, proceed
  - Let me review the links first (pause and wait)
  - Cancel

### Step 7: Apply Confluence Updates

Execute user-approved changes:

> **RITICAL: Section-Only Updates**
>
> When updating a Confluence page, you MUST:
> 1. **Fetch the page in storage format** (set `convert_to_markdown: false`) to get the raw HTML/XML
> 2. **Copy the ENTIRE existing content exactly as-is**
> 3. **Only modify the specific sections** that need changes — leave all other sections completely untouched
> 4. **Use `content_format: storage`** when calling confluence_update_page to preserve the original format
>
> **NEVER:**
> - Rewrite the entire page content
> - Convert storage format to markdown and back — this destroys Jira macros, smart links, and special formatting
> - Touch sections that don't need updates, even if they "look the same"
>
> Confluence pages contain special macros (e.g., `<ac:structured-macro ac:name="jira">`) that will be **permanently broken** if you rewrite them. Always preserve the original XML structure for unchanged sections.

1. Use confluence_update_page to update **only the affected sections** of each document — do NOT overwrite unrelated content
2. Use confluence_add_comment to add a change log comment with this format:

```
📋 Auto-updated by Document Orchestrator

Meeting: {meeting title/date}
Changes applied:
- {section}: {brief description of change}
- {section}: {brief description of change}

Reason: Based on decisions from {meeting date} meeting
To revert: Use Confluence page history (⋯ menu → Page history)
```

### Step 8: Completion Report

Summarize the update results:
- List of updated documents with direct links
- Summary of changes per document
- Rollback instructions: "If any update is incorrect, go to the page → click ⋯ → Page history → Restore previous version"
- Any failed updates with error details

## Audio Transcription

When the user provides an audio file, use GPT-4o Transcribe for speech-to-text, then Claude analyzes the result.

**Requirements:**
- `ffmpeg` must be installed (for audio duration detection and chunking long files)
- `openai` Python package (installed via `pip install openai`)

**Long audio support:** The script automatically handles audio files longer than 23 minutes by splitting them into chunks using ffmpeg, transcribing each chunk separately, and combining the results.

1. Run the transcription script (always save to logs/ directory):
   ```
   python scripts/transcribe.py <audio_file_path> logs/YYYY-MM-DD_HH-MM_transcript.json
   ```
2. The script outputs a JSON file with:
   - `transcript`: Raw full text transcription
   - `source_file`: Original audio file path
   - `duration_seconds`: Total audio duration
   - `processed_at`: Timestamp
   - `chunks`: (only for long audio) Number of chunks and chunk duration
3. Read the transcript JSON via the Read tool
4. Claude then analyzes the raw transcript to extract decisions, action items, and keywords (same as text input flow)

If the script is not available or fails, ask the user to provide a text transcript instead.

## Logging

Record all update activities in the project's logs/ directory:
- Filename: YYYY-MM-DD_HH-MM_update.json
- Contents:
  ```json
  {
    "timestamp": "ISO 8601",
    "meeting_source": "file path or 'pasted text'",
    "meeting_title": "Meeting title or date",
    "meeting_summary": "2-3 sentence summary of the meeting",
    "attendees": ["Name (Role)", ...],
    "decisions": [
      "Decision 1 description",
      "Decision 2 description"
    ],
    "documents_analyzed": [
      {
        "page_id": "123456789",
        "title": "Page Title",
        "url": "https://nota-dev.atlassian.net/wiki/...",
        "owner": "Owner name",
        "last_updated": "ISO 8601"
      }
    ],
    "changes_proposed": [
      {
        "page_id": "123456789",
        "title": "Page Title",
        "url": "https://nota-dev.atlassian.net/wiki/...",
        "section": "Section name",
        "classification": "REQUIRED or RECOMMENDED",
        "description": "What was changed and why"
      }
    ],
    "changes_applied": [
      {
        "page_id": "123456789",
        "title": "Page Title",
        "url": "https://nota-dev.atlassian.net/wiki/...",
        "section": "Section name",
        "description": "What was changed",
        "version": "1 -> 2"
      }
    ],
    "changes_skipped": [
      {
        "page_id": "123456789",
        "title": "Page Title",
        "section": "Section name",
        "reason": "Why it was skipped"
      }
    ],
    "errors": [
      {
        "page_id": "123456789",
        "title": "Page Title",
        "error": "Error description"
      }
    ]
  }
  ```

## Important Constraints

- **Section-level edits only**: Never replace an entire page. Only modify the specific sections that need changes.
- **Always show links**: When referencing any Confluence page, always include the full clickable URL.
- **User confirmation required**: Never update a page without explicit user approval.
- **Preserve formatting**: When updating a section, maintain the existing page formatting (headings, tables, etc.).

## Critical: Confluence Update Rules

**DO NOT rewrite pages. ONLY patch specific sections.**

Confluence pages contain complex XML structures including:
- Jira ticket macros (`<ac:structured-macro ac:name="jira">`)
- Smart links with card appearances
- Emoji macros (`<ac:emoticon>`)
- Local IDs for collaborative editing
- Nested list structures with preserved IDs

**Correct update procedure:**
1. Fetch page with `convert_to_markdown: false` to get storage (HTML/XML) format
2. Identify the exact HTML block for the section you need to update
3. Replace ONLY that block, keeping everything else byte-for-byte identical
4. Submit with `content_format: storage`

**Example - updating only Action Items section:**
```
# Original content (storage format)
...<h2>Action items</h2><ac:task-list>...</ac:task-list><h2>Decisions</h2>...

# Replace ONLY the task-list block:
...<h2>Action items</h2><ol><li>New item 1</li><li>New item 2</li></ol><h2>Decisions</h2>...
```

**Why this matters:** If you rewrite sections that weren't meant to change, Jira macros become plain text, smart links break, and collaborative editing metadata is lost. Users will need to manually restore the page from version history.

## MCP Tool Limitations

The Confluence MCP tools do NOT support:
- **Page version history** - Cannot fetch previous versions
- **Restore to version** - Cannot programmatically revert changes

If an update breaks a page, the user must manually restore via Confluence UI:
1. Go to the page → click ⋯ menu → Page history
2. Find the previous version → click Restore
