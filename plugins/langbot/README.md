# Langbot for Claude Code

```
/plugin marketplace add langbot-ai/claude-plugins
/plugin install langbot@langbot
```

Then ask Claude "what can I do with Langbot?". The plugin adds:

- the Langbot MCP server (`https://mcp.langbot.ai/mcp`, signs you in to Langbot in the browser on
  first use; no token to copy), and
- the `langbot` skill: Claude finds a strategy for a topic you follow, tests it on recorded
  markets with Kalshi's fees and realistic delays, explains the results plainly (most ideas lose
  after fees), runs it on paper against live markets, and, when you ask, takes it live on your own
  Kalshi account right from the chat: give it your Kalshi key and a budget; Langbot uses the key
  once, deletes it on Kalshi and runs the bot on a new trade-only key.

Docs: https://langbot.ai/docs/claude-plugin
