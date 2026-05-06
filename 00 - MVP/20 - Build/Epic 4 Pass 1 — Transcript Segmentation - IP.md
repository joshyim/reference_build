# Epic 4: Pass 1 — Transcript Segmentation - BE

This plan outlines the implementation for segmenting transcripts into topical blocks, focusing on data persistence and idempotency. The goal is to ensure each pipeline run can parse transcripts via LLM (using out-of-the-box BAML configurations that already instruct the model to filter intro banter and tangents), construct the structured segments rows, and accurately upsert them into the database.

User Review Required

[!IMPORTANT]
A small schema migration is required: The segments table currently lacks a summary column, even though it's requested in the Epic 4 acceptance criteria and produced by BAML (TopicSegment.summary). The plan proposes a new SQL migration file (003_add_summary_to_segments.sql) to append this column.

[!NOTE] 
The requirement to "filter out intro banter, logistics, and sub-30-second tangents" is currently fulfilled by the system prompt specified in backend/baml_src/transcript_chunking.baml. No BAML changes are planned unless further refinement is needed.
Per your feedback, both source_id and episode_title are out of scope for passing down from the API request or persisting in this step, so no API schema updates are necessary.

Proposed Changes

Storage Schema

[NEW] backend/sql/003_add_summary_to_segments.sql

Create a migration file to run ALTER TABLE segments ADD COLUMN IF NOT EXISTS summary TEXT;.

Step Execution logic

[MODIFY] backend/app/pipeline/steps/segmentation.py

In execute():

Add summary to the INSERT INTO segments SQL parameters.

Refactor the persistence logic to be idempotent: change ON CONFLICT (id) DO NOTHING to ON CONFLICT (id) DO UPDATE SET so that if we rerun Pass 1 on the exact same episode_id + segment_number (which determines the primary key id), we seamlessly overwrite the prior segment data.

In load_persisted_results():

Also SELECT summary from segments and map it back into TopicSegment(summary=row["summary"]) for hydration.

Open Questions

None at this time. All epic requirements and feedback are accounted for.

Verification Plan

Automated Tests

Create a backend/tests/ folder and store a new test file, e.g., test_segmentation.py.

Restart the backend to let FastAPI reload and apply schema changes.

Test hitting POST /pipeline/run with transcript_url (without source_id or episode_title) and track the status of segmentation inside the tests.

Verify the DB tables pipeline_runs and segments receive the correct attributes, especially checking the populated summary column in segments.

Manual Verification

Re-run the endpoint with force=True using the same episode_id and confirm that no duplicate rows are generated and changes reflect the latest LLM output (thanks to idempotent UPSERT).