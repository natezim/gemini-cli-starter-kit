name = "setup"
description = "First-time setup wizard — interviews the user and builds CONTEXT.md"
prompt = """
A new user is setting up this Gemini CLI workspace for the first time.

Introduce yourself:
"Welcome. I'll ask you a few questions to build your project context file.
This takes about 5 minutes and means you won't have to re-explain things every session.
I'll go one question at a time."

Ask each question ONE AT A TIME. Confirm each answer before moving on.

1. "What are we working on? Give me a 2-3 sentence description of this project or workspace."
2. "What tools and systems will we be using? For example: databases, APIs, CLIs, cloud platforms, file systems."
3. "How do you connect or authenticate to those systems?"
4. "Are there specific tables, files, APIs, or data sources we'll use regularly?"
5. "What are the recurring tasks you do most often in this workspace?"
6. "Any known gotchas or constraints I should always remember — quirks, things that have caused problems, things to never touch?"
7. "Any naming or formatting conventions for output files?"
8. "Are there cost or safety limits where I should stop and ask before proceeding?"
9. "What does 'done' look like for a typical task here?"
10. "What are you working on right now specifically?"

Once all answers are collected:
- Write a complete ./CONTEXT.md with their actual answers. No placeholder text.
- Leave a section blank only if they had nothing to say about it.
- Show a summary and ask: "Does this look right? Anything to add or change?"
- Apply any corrections.
- Say: "CONTEXT.md is ready. Run /init to start your first session."
"""
