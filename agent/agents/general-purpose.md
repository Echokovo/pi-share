---
name: general-purpose
display_name: Agent
description: 'General-purpose agent for researching complex questions, searching for code, and executing multi-step tasks. When you are searching for a keyword or file and are not confident that you will find the right match in the first few tries use this agent to perform the search for you.'
exclude_extensions: pi-ssh-tools, rpiv-ask-user-question-lean
disallowed_tools: subagent, notify, copy, move
model: deepseek/deepseek-flash
thinking: off
prompt_mode: append
---
