# Writing a new harness adapter

`src/` is the portable compaction engine. It has no Claude Code types in it:
`Message`, `ToolUse`, and `ToolResult` (in `src/types.ts`) are plain shapes, and the
TypeSafe/Jev call sits behind the `JevAsker` interface (`ask(state, questions)`), not
hardcoded into the engine. `hooks/fast-jev.ts` is the one adapter that exists today,
for Claude Code. This document describes the contract any other adapter needs to
meet, so adding one is a bounded task instead of a rewrite.

As of this writing, no other agent harness is known to expose a hook that can
intercept or replace context compaction the way Claude Code's `session.compact`
function hook does. This document has nothing to point a second adapter at yet — it
exists so the next person who finds such a hook does not have to reverse-engineer
the contract from `hooks/fast-jev.ts` first.

## What an adapter must do

1. **Build a `Message[]`** from the host harness's own transcript format.
   Each `Message` needs: `role` (`'user' | 'assistant'`), `text`, `toolUses`
   (`ToolUse[]`), and optionally `toolResults` (`ToolResult[]`). A `ToolUse` and its
   matching `ToolResult` are paired by `tool_use_id` — the engine does that pairing
   itself (see `collectToolCalls` in `src/state.ts`), so the adapter only needs to
   supply both sides with matching ids.

2. **Call the engine.** Either:
   - `compact(messages, asker, options)` from `src/compact.ts`, passing your own
     `JevAsker` implementation, or
   - `compactMessages(messages, options)` from `src/messages.ts`, which builds a
     `JevClient` (the TypeSafe/Jev HTTP client in `src/client.ts`) from `options` or
     the `TYPESAFE_API_KEY` environment variable.

   `options` (`CompactOptions` in `src/types.ts`) covers `keepThreshold`,
   `preserveRecentMessages`, `maxStateTokens`, `maxRequestTokens`,
   `truncateHeadChars`, and an optional `goal` string — all have defaults in
   `DEFAULT_OPTIONS` (`src/compact.ts`).

3. **Map `CompactResult.messages` back onto the host's own session format.**
   `hooks/fast-jev.ts`'s `toSessionMessages()` shows the pattern: messages the
   engine returned unchanged are matched back to their original host objects by
   identity, so the host's other metadata on that message survives untouched; only
   messages the engine actually rebuilt (something dropped or truncated) need a
   fresh host-native object built from the engine's plain fields.

4. **Provide a `JevAsker`**, if not using the default `JevClient`. It is one method:
   `ask(state: JevState, questions: JevQuestions): Promise<JevResponse>`. The
   default `JevClient` posts to `https://api.typesafe.ai/v1/systemone` (`src/request.ts`);
   an adapter only needs a custom `JevAsker` if the host harness has its own HTTP
   transport it must route through, the way `hooks/fast-jev.ts`'s `jevAsker()` wraps
   Claude Code's `$.http.fetch`.

5. **Decide when to run it.** The Claude Code adapter hooks `session.compact`
   (replace the built-in summary) and `turn.complete` (trigger compaction at a
   configured context-percent). A different harness's hook surface, if one exists,
   decides this differently — there is nothing in the engine that assumes either
   trigger.

## What NOT to change

- Don't move or rename `hooks/`. Claude Code's plugin loader expects
  `hooks/hooks.json` at that path.
- Don't add harness-specific types into `src/`. If an adapter needs a type only that
  harness understands, it belongs in the adapter's own directory, importing from
  `src/` — never the other way round.
