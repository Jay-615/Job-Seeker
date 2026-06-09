---
name: draft-tailored-resume-coverletter
description: Generate a tailored resume and cover letter for a specific scored job. Takes a rank or job id, reads the scored + inbox files and the user's base resume, writes markdown drafts under applications/<company>-<role>-<date>/, asks the user whether to also produce DOCX via pandoc (default — markdown only on first pass), and writes a notes.md flagging stretches, gaps, and strategic suggestions. Will not fabricate experience the user doesn't have. Does NOT touch the actual job application — the user submits themselves.
---

<!-- Implementation pending — module M5. -->
