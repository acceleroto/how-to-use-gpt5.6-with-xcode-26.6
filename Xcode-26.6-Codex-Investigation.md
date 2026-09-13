# Xcode 26.6 + OpenAI Codex: Investigation Notes

## Purpose

This document captures our investigation into using newer OpenAI Codex
models inside Xcode 26.6's built-in Codex integration while retaining
Xcode-native tools, ChatGPT subscription authentication, and no API key.

## Current result

The best known working combination is:

``` text
Xcode 26.6
Codex 0.148.0
GPT-5.6 Luna
xhigh reasoning
ChatGPT subscription authentication
Xcode-native tools
no service_tier override
```

GPT-5.6 Luna `max` works in Codex 0.148.0 itself, including through
`codex app-server`, but Xcode 26.6 appears unable to represent or parse
`max`. GPT-6 Astra requires Codex 0.153.0 or newer, while the tested
0.153 and 0.154 runtimes currently break Xcode's Account/auth integration.

## Tested environment

-   Xcode 26.6
-   Xcode app-server client: `26.6 (17F112)`
-   Runtime selector path uses `XcodeVersions/17F113`
-   macOS 26.3.1(a)
-   Apple Silicon
-   ChatGPT Plus authentication
-   SwiftUI iOS test project
-   No OpenAI API key

Important paths:

``` text
Xcode-private Codex home:
~/Library/Developer/Xcode/CodingAssistant/codex

Xcode-private config:
~/Library/Developer/Xcode/CodingAssistant/codex/config.toml

Model cache:
~/Library/Developer/Xcode/CodingAssistant/codex/models_cache.json

Codex logs:
~/Library/Developer/Xcode/CodingAssistant/codex/logs_2.sqlite

Version-selected runtime:
~/Library/Developer/Xcode/CodingAssistant/Agents/XcodeVersions/17F113/codex

Installed runtimes:
~/Library/Developer/Xcode/CodingAssistant/Agents/codex/
```

## Stock Xcode 26.6

Xcode originally used Codex 0.140.0. Its model picker exposed Default
and GPT-5.5. The original model cache did not contain GPT-5.6 Luna.

Putting this in Xcode's private config:

``` toml
model = "gpt-5.6-luna"
model_reasoning_effort = "max"
```

produced Xcode's generic `Your request couldn't be completed` error.

## Custom ACP route

We tested `@agentclientprotocol/codex-acp`. Directly, it authenticated
through ChatGPT and could run Luna Max. A direct test returned `LUNA_OK`
with GPT-5.6 Luna at Max reasoning.

The custom ACP agent nevertheless failed inside Xcode across multiple
adapter versions and minimal configurations. Because the goal is
Xcode-native integration and tools, this route was abandoned.

## Swapping Xcode's Codex runtime

Xcode launches a separately installed Codex runtime through:

``` text
~/Library/Developer/Xcode/CodingAssistant/Agents/XcodeVersions/17F113/codex
```

This allowed newer Codex versions to be tested without modifying Xcode
itself.

### Codex 0.147.0

0.147 authenticated in Xcode and exposed GPT-5.6 Luna. Its regenerated
model catalog contained Luna reasoning levels through `max`.

Direct CLI Luna Max worked. In Xcode:

  Configuration         Result
  --------------------- --------
  Luna high             Works
  Luna xhigh            Works
  Luna max              Fails
  Direct CLI Luna max   Works

### Critical discovery: `codex-code-mode-host`

Initially only `codex` was copied into newer runtime directories. Newer
Codex versions also expect the matching sibling executable:

``` text
codex-code-mode-host
```

The logs showed:

``` text
failed to spawn code-mode host .../codex-code-mode-host:
No such file or directory (os error 2)
```

The correct 0.148 runtime layout is:

``` text
0.148.0/
  codex
  codex-code-mode-host
```

Both binaries must come from the same release. Adding the helper fixed
Xcode-native tool use.

### Codex 0.148.0: known-good endpoint

With both matching binaries, all of these work:

-   Xcode Account authentication
-   ChatGPT subscription authentication
-   GPT-5.6 Luna
-   `xhigh` reasoning
-   Xcode-native tools
-   project navigator inspection
-   source editing through Xcode tools
-   project builds
-   no API key
-   no custom ACP agent

Known-good private config:

``` toml
model = "gpt-5.6-luna"
model_reasoning_effort = "xhigh"
```

Useful tool-only verification prompt:

``` text
Use the Xcode tools to tell me the currently open project's name and list its top-level project files. Do not use shell commands or direct filesystem inspection.
```

Stronger edit test:

``` text
Using only the Xcode tools, edit ContentView.swift and change the displayed text in the default SwiftUI view to "Xcode Tools Are Working". Do not use shell commands or direct filesystem access. After making the change, tell me which Xcode tool you used.
```

Both worked. Xcode also launches its native bridge:

``` text
/Applications/Xcode.app/Contents/Developer/usr/bin/mcpbridge
```

### Codex 0.149.0

0.149 failed the Xcode baseline with the generic request error. Public
reports also described 0.149 app-server/TUI regressions that were absent
in 0.148. It is not recommended for this setup.

### Codex 0.154 alpha and release

We tested ChatGPT's `0.154.0-alpha.6.2` and public release `0.154.0`.
The release contained both required binaries and both were staged
correctly.

With either runtime, Xcode showed:

``` text
Codex Requires Eligible Account or API Key
```

The Account row spun indefinitely and no usable model selection was
available. Rolling back to 0.148 restored normal operation.

This failure happens before GPT-6 Astra can be tested.

## GPT-6 Astra

OpenAI's Codex model information indicates GPT-6 Astra requires Codex
0.153.0 or newer. Its model ID is:

``` text
gpt-6-astra
```

This leaves the current compatibility gap:

``` text
Xcode 26.6 + Codex 0.148
  -> works
  -> too old for Astra

Xcode 26.6 + Codex 0.154
  -> new enough for Astra
  -> Xcode Account/auth integration fails
```

## Service tier findings

With Xcode + Codex 0.148:

  `service_tier`   Result
  ---------------- --------
  omitted          Works
  `"flex"`         Works
  `"default"`      Fails
  `"priority"`     Fails
  `"fast"`         Fails

Codex source associates Fast with the request value `priority` and
accepts `fast` or `priority` when parsing, but Xcode did not
successfully use those values.

Current recommendation: omit `service_tier`.

## Max reasoning: decisive findings

The major question was whether Max failed because of Codex, the backend,
authentication, or Xcode.

### Genuine Codex 0.148 app-server runs Luna Max

We manually drove the genuine 0.148 `codex app-server` JSON protocol.

A `thread/start` specifying:

``` json
{
  "model": "gpt-5.6-luna",
  "reasoningEffort": "max"
}
```

succeeded.

We then issued a real turn asking:

``` text
Reply with exactly MAX_APP_SERVER_OK
```

The app-server accepted Max, emitted reasoning, returned exactly
`MAX_APP_SERVER_OK`, reported reasoning output tokens, completed
normally, and authenticated through ChatGPT Plus.

Therefore:

> Codex 0.148.0 + app-server + ChatGPT Plus + GPT-5.6 Luna + Max works
> end-to-end.

This rules out the Luna backend, account, ChatGPT subscription auth,
Codex 0.148, and app-server as the cause of Xcode's Max failure.

### Xcode 26.6's `ReasoningEffort` enum

The relevant Xcode framework is:

``` text
/Applications/Xcode.app/Contents/PlugIns/IDEIntelligenceAgents.framework/Versions/A/IDEIntelligenceAgents
```

Swift symbol inspection identified:

``` text
IDEIntelligenceAgents.Codex.ReasoningEffort
```

with cases corresponding to:

``` text
none
minimal
low
medium
high
xhigh
```

There is no `max` case.

Disassembly of `ReasoningEffort.init(rawValue:)` showed a six-case
string switch. Unknown input falls outside the recognized cases.

This is strong evidence that:

> Xcode 26.6 itself cannot represent the Codex `max` reasoning value.

No Xcode binary patch was attempted or desired.

### Failure boundary

During a failed Max request:

1.  Xcode launches genuine Codex 0.148 successfully.
2.  Codex launches Xcode's `mcpbridge`.
3.  MCP initialization succeeds.
4.  Codex exits almost immediately.
5.  No normal conversation turn reaches the Codex log database.

During xhigh, the process continues, connects to Xcode tools, invokes
tools, and completes.

## Clean workarounds already tested

### Hot-swap config

Start Xcode with xhigh, change the private config to Max while Xcode
remains open, then submit a request.

Result: failure.

Xcode apparently re-reads or re-resolves configuration when starting the
conversation.

### Project `.codex/config.toml`

Xcode-private config was xhigh while the project-level
`.codex/config.toml` requested Luna Max.

Result: failure.

The project config does not bypass Xcode's parser.

### Executable wrapper/proxy

A shell wrapper and compiled native Mach-O proxy were tested to observe
or alter Xcode-to-Codex communication. Both worked manually but Xcode
did not execute them in place of the expected Codex binary.

This suggests some executable validation or integrity check.
Code-signature validation is plausible but was not directly proven.

The genuine 0.148 binary was restored.

## Xcode uses a private `CODEX_HOME`

The user's normal Codex/Desktop config at:

``` text
~/.codex/config.toml
```

already contains Luna Max. Yet Xcode follows its own private xhigh
config.

Inspecting the live Xcode-launched Codex process revealed why:

``` text
CODEX_HOME=/Volumes/External1/Users/bryan/Library/Developer/Xcode/CodingAssistant/codex
HOME=/Volumes/External1/Users/bryan
```

Generically:

``` text
CODEX_HOME=~/Library/Developer/Xcode/CodingAssistant/codex
```

So Xcode deliberately gives Codex a private configuration home. The
normal `~/.codex/config.toml` does not control Xcode's Codex runtime.

## Latest important log result

A successful xhigh request showed:

``` text
ThreadSettingsOverrides {
    ...
    model: None,
    effort: None,
    ...
    service_tier: None
}
```

Xcode therefore does **not** appear to send `effort: xhigh` as a
per-turn override.

Codex later resolves:

``` text
codex.turn.reasoning_effort=xhigh
```

and sampling reports:

``` text
effort=Some(XHigh)
auth_mode=Some(Chatgpt)
```

This is important because xhigh is being resolved from configuration
rather than explicitly supplied by Xcode with each turn.

## Likely architecture

The evidence currently fits:

``` text
Xcode 26.6
    |
    | sets CODEX_HOME to Xcode-private Codex directory
    |
    | reads/parses Codex configuration using
    | IDEIntelligenceAgents.Codex.ReasoningEffort
    | whose supported cases stop at xhigh
    |
    +--> codex 0.148 app-server
            |
            | reads/resolves configuration
            |
            +--> GPT-5.6 Luna
```

For xhigh, both Xcode and Codex understand the value.

For Max, Codex understands it, but Xcode does not.

## Next research steps

The next step should be to inspect the **Codex 0.148 source
specifically** for a configuration mechanism with higher precedence than
`config.toml`.

The ideal workaround would be:

``` text
Xcode-private config.toml:
    model_reasoning_effort = "xhigh"

Xcode parses:
    xhigh

Higher-precedence Codex-only override:
    max

Actual Luna request:
    max
```

Things to investigate:

1.  Environment-variable config overrides in Codex 0.148.
2.  Runtime/app-server configuration layers above `config.toml`.
3.  Whether Xcode preserves arbitrary inherited environment variables
    when launched from Terminal.
4.  Whether Xcode only validates TOML or transforms configuration before
    Codex consumes it.

The 0.148 release corresponds to:

``` text
rust-v0.148.0
commit 3ba0f71
```

If no clean higher-precedence override exists, the next option is
observing the genuine signed app-server protocol without replacing its
executable. Possible approaches include Codex debug/OTEL
instrumentation, debugger attachment, or OS-level stdin/stdout tracing.
macOS hardened runtime and SIP may limit these approaches.

Longer term, re-test new Codex and Xcode releases. A future Codex
release may restore Xcode authentication while supporting Astra, and a
future Xcode release may add `max` to Apple's `ReasoningEffort` enum.

## Useful commands

Check active runtime:

``` bash
readlink "$HOME/Library/Developer/Xcode/CodingAssistant/Agents/XcodeVersions/17F113/codex"
```

Check runtime files:

``` bash
ls -la "$HOME/Library/Developer/Xcode/CodingAssistant/Agents/XcodeVersions/17F113/codex/"
```

Check version:

``` bash
"$HOME/Library/Developer/Xcode/CodingAssistant/Agents/XcodeVersions/17F113/codex/codex" --version
```

Inspect Xcode-launched Codex environment while a request is active:

``` bash
ps eww -axo pid,ppid,command |
grep '[X]codeVersions/17F113/codex/codex app-server'
```

Inspect recent reasoning logs:

``` bash
DB="$HOME/Library/Developer/Xcode/CodingAssistant/codex/logs_2.sqlite"

sqlite3 -header -column "$DB" "
SELECT
  ts,
  target,
  feedback_log_body
FROM logs
WHERE feedback_log_body LIKE '%ThreadSettingsOverrides%'
   OR feedback_log_body LIKE '%reasoning_effort%'
ORDER BY ts DESC, ts_nanos DESC, id DESC
LIMIT 30;
"
```

Inspect Xcode reasoning symbols:

``` bash
BIN="/Applications/Xcode.app/Contents/PlugIns/IDEIntelligenceAgents.framework/Versions/A/IDEIntelligenceAgents"

nm -m "$BIN" 2>/dev/null |
xcrun swift-demangle |
grep -i 'ReasoningEffort'
```

Search reasoning-level strings:

``` bash
BIN="/Applications/Xcode.app/Contents/PlugIns/IDEIntelligenceAgents.framework/Versions/A/IDEIntelligenceAgents"

strings "$BIN" |
grep -x -E 'none|minimal|low|medium|high|xhigh|max|ultra|maximum'
```

## Known-good configuration

``` toml
model = "gpt-5.6-luna"
model_reasoning_effort = "xhigh"
```

Do not currently add a `service_tier` override.

Also remove any stale project-level Max test config such as:

``` text
<Project>/.codex/config.toml
```

## Runtime installation reminder

For newer Codex releases, do not copy only `codex`. Copy the matching
helper too:

``` text
codex
codex-code-mode-host
```

Example:

``` text
~/Library/Developer/Xcode/CodingAssistant/Agents/codex/0.148.0/
    codex
    codex-code-mode-host
```

After changing runtimes, moving `models_cache.json` aside and fully
relaunching Xcode can regenerate the catalog. Do not manually fabricate
model-cache entries.

## Current recommendation

For real Xcode work today, use Codex 0.148.0, GPT-5.6 Luna, xhigh
reasoning, ChatGPT subscription authentication, Xcode-native tools, and
no service-tier override.

The central finding is:

> **GPT-5.6 Luna Max is functional in genuine Codex 0.148 app-server
> with ChatGPT Plus. Xcode 26.6 is the component preventing it, and
> Xcode's own six-case `ReasoningEffort` enum provides a concrete
> technical explanation.**

That gives the next investigation a much narrower target: find a
Codex-only configuration layer above Xcode's private TOML, or wait for
Xcode to support the newer reasoning level directly.
