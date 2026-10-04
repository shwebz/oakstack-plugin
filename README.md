# Oakstack for Claude Code

Lets Claude add **scheduled jobs** and a **webhook relay** to your app with [Oakstack](https://oakstack.dev):

- "Run my cleanup route every night at 3am Chicago time."
- "Put Stripe webhooks behind something that never loses events."
- "Why did last night's webhook fail? Replay it."

This plugin includes:

- **The Oakstack skill**, which teaches Claude when to use Oakstack and how to wire it into your code: the `oakstack` npm package, verifying signatures, testing, and the gotchas.
- **The hosted Oakstack MCP server** (`https://oakstack.dev/mcp`), so Claude can create and debug jobs and webhook endpoints directly.

## Install

1. Get an API key at [oakstack.dev/dashboard/api-keys](https://oakstack.dev/dashboard/api-keys).
2. Make it available to Claude Code by adding it to your shell profile (`~/.zshrc` or `~/.bashrc`), then open a new terminal:

   ```sh
   export OAKSTACK_API_KEY=ok_live_...
   ```

3. In Claude Code:

   ```
   /plugin marketplace add shwebz/oakstack-plugin
   /plugin install oakstack@oakstack
   ```

4. Restart Claude Code. Then ask Claude for what you need, for example "add a nightly cleanup job with Oakstack".

## Without the plugin

- Skill only: `mkdir -p ~/.claude/skills/oakstack && curl -fsSL https://oakstack.dev/SKILL.md -o ~/.claude/skills/oakstack/SKILL.md`
- MCP only: `claude mcp add --transport http oakstack https://oakstack.dev/mcp --header "Authorization: Bearer $OAKSTACK_API_KEY"`
- SDK: `npm install oakstack`

Docs: [oakstack.dev/docs](https://oakstack.dev/docs) · For agents: [oakstack.dev/llms.txt](https://oakstack.dev/llms.txt)

This repo is generated from the main Oakstack repository; please open issues rather than pull requests.
