---
name: feedback-communication
description: How the user wants Claude to communicate — concise, guided, no hand-holding, no narrating understanding
metadata:
  type: feedback
---

Keep responses concise. User explicitly asked for "concise response" multiple times across sessions.
**Why:** User finds long explanations noisy; they prefer to ask follow-up questions themselves.
**How to apply:** Default to short answers. One concept at a time. Only expand if they ask.

Do not narrate your understanding back to them before acting.
**Why:** User said "read it, no need to state your understanding" twice across sessions.
**How to apply:** When asked to read a file and then do something — just do it. Skip the "I can see that..." preamble.

Use Q&A / walkthrough style for learning tasks, not spoon-feeding.
**Why:** User explicitly requested "walkme through, like Q&A style" and "do not do it for me, help on the sideline."
**How to apply:** For learning-oriented tasks, ask one question at a time. Guide, don't implement.

Use structured prompts when the user has a complex request.
**Why:** User naturally structures their own messages with [Role], [Context], [Objective], [Task], [Remark] sections.
**How to apply:** Match that structure when responding to structured requests; it shows you read them carefully.

Do not destructively modify the existing `frontend/` and `backend/` source.
**Why:** They are the reference implementation for the redesign exercise.
**How to apply:** Any new code goes in a separate location; treat the existing code as read-only unless explicitly asked to change it.