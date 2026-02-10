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

**Required environment variables in `.env`:**
```
CONFLUENCE_URL=https://your-domain.atlassian.net
CONFLUENCE_EMAIL=your-email@example.com
CONFLUENCE_TOKEN=your-api-token
OPENAI_API_KEY=your-openai-key
```

> **Note:** The `.mcp.json` maps these to the variable names each MCP server expects.
> Do NOT check for `CONFLUENCE_USERNAME` or `CONFLUENCE_API_TOKEN` directly — those are
> internal mappings handled by `.mcp.json`.

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

After reading the meeting notes, store the raw transcript text for classification in the next steps.

### Step 2.5: Classify Meeting

Classify the meeting transcript using the two lightweight classifiers. These are pure prompt-based — no page fetching.

1. Read `domains/classifiers/topic.md` (relative to this SKILL.md file)
2. Apply the topic classifier to the raw transcript → outputs `topics: [...]` (1-2 topics from: Weekly Progress, Sprint Planning, Scenario & Product, Technical Design, Experiment & Validation)
3. Read `domains/classifiers/part.md`
4. Apply the part classifier to the raw transcript → outputs `parts: [...]` (1-3 parts from: Q, GO, MR, ME, SWE)

Store both classification results for the next step.

### Step 2.6: Domain-Aware Analysis

For each classified part, load its specialized agent and analyze the transcript.

1. Read `domains/cross-cut/scenarios.md` — this shared S0-S3 context is always loaded
2. For each part in the classification result:
   a. Read the corresponding agent file: `domains/agents/{part_code}.md` (where part_code is one of: q, go, mr, me, swe — lowercase)
   b. Analyze the transcript from that part's perspective using:
      - The agent's **baked-in domain glossary** to correctly interpret terminology
      - The **per-topic focus** instructions matching the classified topics
      - The **S0-S3 cross-cut context** from scenarios.md
   c. Based on the classified topics, identify which **reference pages** to fetch from the agent's reference table
   d. Fetch relevant reference pages via `confluence_get_page` (page IDs listed in the agent file)
   e. Use the reference page content to enrich the analysis (verify current state of docs, compare with decisions)
   f. Produce part-specific output: decisions, action items, keywords, reference pages, cross-cut impacts

### Step 2.7: Combine & Cross-Cut Check

Merge the outputs from all activated part agents into a unified analysis.

1. **Merge decisions**: combine all part-specific decisions, noting which part identified each
2. **Merge action items**: combine and deduplicate (same action may be detected by multiple parts)
3. **Merge keywords**: combine domain-specific search keywords from all parts
4. **Merge reference pages**: collect all reference page IDs that were fetched or recommended
5. **Cross-cut check**: review all `cross_cut_impacts` from part agents:
   - If any decision affects S0-S3 scenario definitions → flag for scenario doc updates
   - If any decision affects the shared pipeline flow → flag for pipeline doc updates
   - If any decision affects multiple parts → flag for cross-part coordination
6. Produce unified output:
   - Combined decisions list
   - Combined action items list
   - Combined search keywords (domain-aware, not generic)
   - Combined reference page IDs (already fetched)
   - Cross-cut impact summary

### Step 3: Find Related Confluence Documents

Search Confluence for documents related to the combined keywords from the domain-aware analysis. All searches are scoped to the **NPP02** (NetsPresso Platform Team) space.

1. Use confluence_search with keyword-based queries, scoped to `space=NPP02`:
   - Use the domain-specific keywords from Step 2.7 (not generic meeting keywords)
   - Run multiple queries for different decision topics
   - Include reference page IDs from part agents as known-relevant pages
2. Merge search results with:
   - Pinned documents from Step 1 (if any)
   - Reference pages already fetched in Step 2.6
   - Deduplicate by page ID
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

Execute user-approved changes using the **confluence-helper MCP tools**:

**Use `mcp__confluence-helper__confluence_patch_section`** to update specific sections:
```
confluence_patch_section(
    page_id="123456789",
    section_title="Action items",      # The heading text to find
    new_content="<ol><li>...</li></ol>", # New HTML content (without the heading)
    version_comment="Updated from meeting 2026-02-04"
)
```

This tool automatically:
- Fetches the page in storage format
- Finds the section by heading title
- Replaces only that section's content
- Preserves all other content exactly as-is

> **Why use confluence_patch_section instead of confluence_update_page?**
>
> The patch tool guarantees section-only updates. Using `confluence_update_page` directly
> risks accidentally rewriting the entire page and breaking Jira macros, smart links, etc.

After updating, use confluence_add_comment to add a change log comment with this format:

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
- Previous version numbers (so user can request revert if needed)
- Any failed updates with error details

**If the user wants to revert changes**, use `mcp__confluence-helper__confluence_restore_version`:
```
confluence_restore_version(
    page_id="123456789",
    version=11,  # The version number before your changes
    message="Reverted: user requested rollback"
)
```

This is preferred over telling the user to manually revert via Confluence UI.

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

> **Note:** The `confluence_patch_section` helper tool handles this automatically.
> This section explains the underlying issue for context.

**DO NOT rewrite pages. ONLY patch specific sections.**

Confluence pages contain complex XML structures including:
- Jira ticket macros (`<ac:structured-macro ac:name="jira">`)
- Smart links with card appearances
- Emoji macros (`<ac:emoticon>`)
- Local IDs for collaborative editing
- Nested list structures with preserved IDs

**Why this matters:** If you rewrite sections that weren't meant to change, Jira macros become plain text, smart links break, and collaborative editing metadata is lost.

**Solution:** Use `confluence_patch_section` which handles all of this automatically.

## Helper MCP Tools (confluence-helper)

In addition to the standard `mcp-atlassian` tools, this skill uses **confluence-helper** MCP server (`scripts/helper_mcp.py`) which provides:

| Tool | Description |
|------|-------------|
| `confluence_patch_section` | Update only a specific section by heading title |
| `confluence_get_history` | Get version history for a page |
| `confluence_get_version_content` | Get content at a specific version |
| `confluence_restore_version` | Restore a page to a previous version |

**Always prefer these tools over manual operations:**
- Use `confluence_patch_section` instead of `confluence_update_page` for safer updates
- Use `confluence_restore_version` instead of telling users to manually revert
