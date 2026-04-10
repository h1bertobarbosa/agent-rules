---
description: Create or update an Obsidian-friendly knowledge base note for the project,repository,current task, feature, module, or workflow.
---

<system_instructions>
Instructions:

1. Infer the topic from the current conversation, open files, recent files, git context, and repository structure.
2. If the user did not specify a target path or filename, create a clear kebab-case markdown file in the current workspace root using a name like `knowledge-base-<topic>.md`.
3. Write the note for developers on the team, not for end users.
4. Write in clear Portuguese-br by default, unless the user explicitly asks for another language.
5. Structure the note well. Prefer these sections when relevant:
   - Title
   - Objective
   - Executive Summary
   - Business Context
   - Business Rules
   - Technical Flow
   - API or Contract
   - Data Source or Persistence
   - Operational Notes
   - Troubleshooting
   - Tests and Validation
   - Related Files
   - Future Improvements
6. Include relevant business rules, technical decisions, assumptions, edge cases, and operational caveats whenever they can be inferred.
7. Prefer practical, reusable knowledge over changelog-style text.
8. Use Markdown that works well in Obsidian. You may add lightweight frontmatter with fields like `tags`, `status`, and `feature` when useful.
9. Keep the writing organized, explicit, and easy to scan.
10. If there is not enough context to produce a high-quality note, ask only the minimum necessary clarification.

Output expectations:

- Create the markdown file directly.
- Tell the user the file path that was created.
- Summarize the main sections included.

</system_instructions>
