name = "learnings:add"
description = "Log a discovery to learnings.md immediately — don't wait until session end"
prompt = """
Ask: "What did we just learn? Describe it briefly."

Append to ./output/notes/learnings.md:
  [YYYY-MM-DD HH:MM] <what was learned>
  Context: <where or how it came up>

Show what was appended and confirm.
"""
