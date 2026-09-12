# Using GPT-5.6 Luna and Sol in Xcode 26.6 Codex

This documents a tested way to make Xcode's built-in Codex integration use a newer Codex runtime and newer OpenAI models through a normal ChatGPT subscription. No API key is required.

Heads up: xhigh is the highest reasoning mode that was found to work. If you want Luna max, this won't get you that. Luna xhigh does work though.

## Tested environment

- Xcode Version 26.6 (17F113)
- macOS 26.3.1(a)
- Apple Silicon Mac
- ChatGPT account authentication
- Codex 0.148.0
- GPT-5.6 Luna
- GPT-5.6 Sol

This is not an officially supported Xcode configuration. Xcode updates can change the Codex app-server protocol, paths, or bundled runtime and may require repeating or revising these steps.

## Why do this?

Xcode 26.6 shipped with an older Codex runtime. In the configuration tested here, Xcode was using Codex 0.140.0 and its model picker exposed only `Default` and `GPT-5.5`.

Replacing that runtime with Codex 0.148.0 allows Xcode's existing Codex UI to use newer models while continuing to authenticate with the ChatGPT account already configured in Xcode.

The practical benefits are:

- **GPT-5.6 Luna:** fast coding model with substantially lower ChatGPT subscription usage than heavier models. For routine coding work, this can greatly reduce subscription burn while retaining strong coding performance.
- **GPT-5.6 Sol:** higher-capability model for harder coding and reasoning tasks.
- **Higher reasoning levels:** Luna works at `xhigh` reasoning in this Xcode configuration.
- **No API billing:** authentication remains through the ChatGPT account rather than an OpenAI API key.
- **Stock Xcode workflow:** this keeps Xcode's built-in Codex integration instead of replacing it with a separate ACP agent or external coding tool.

## What was tested

| Configuration | Result |
|---|---|
| Xcode's original Codex 0.140.0 | Works, but did not expose GPT-5.6 Luna |
| Codex 0.147.0 | Works in Xcode |
| Codex 0.148.0 | Works in Xcode and is the recommended tested version |
| Codex 0.149.x | Failed through Xcode |
| Codex 0.154.0-alpha.6.2 | Xcode Codex account handshake failed |
| Codex 0.154.0 release | Xcode Codex Account row spun indefinitely; account initialization failed even with both runtime binaries installed |
| GPT-6 Astra through Xcode 26.6 | Could not be tested because Codex 0.154.0 could not complete Xcode account initialization |
| GPT-5.6 Luna `high` | Works |
| GPT-5.6 Luna `xhigh` | Works |
| GPT-5.6 Luna `max` through Xcode | Fails |
| GPT-5.6 Luna `max` directly from Codex CLI | Works |
| GPT-5.6 Sol `low` on 0.148 | Works |
| GPT-5.6 Sol `medium` on 0.148 | Works |

Luna `max` working directly in the Codex CLI but failing through Xcode indicates that the limitation is in the Xcode/app-server integration path, not Luna itself.

### Service tier testing

| `service_tier` setting | Xcode result |
|---|---|
| omitted | Works |
| `"flex"` | Works |
| `"fast"` | Fails |
| `"priority"` | Fails |
| `"default"` | Fails |

With the setting omitted, Luna uses the normal standard/default service tier. The recommended configuration therefore has **no `service_tier` line**.

Codex 0.148 contains Fast-mode support and the Luna model catalog advertises a Fast/priority tier, but Xcode 26.6 did not successfully enable it in this tested path.

## Installation

### 1. Find Xcode's current Codex runtime

Xcode 26.6 uses a version-specific symlink:

```bash
BASE="$HOME/Library/Developer/Xcode/CodingAssistant/Agents"
XCODE_BUILD="17F113"

readlink "$BASE/XcodeVersions/$XCODE_BUILD/codex"
```

On the tested installation, this pointed to Codex 0.140.0.

Save the original target before changing anything:

```bash
ORIGINAL_CODEX_TARGET="$(readlink "$BASE/XcodeVersions/$XCODE_BUILD/codex")"
echo "$ORIGINAL_CODEX_TARGET"
```

Keep that value if you want an easy rollback later.

### 2. Install Codex 0.148.0

```bash
npm install -g @openai/codex@0.148.0
```

Check the installed version:

```bash
codex --version
```

Find the native Apple Silicon binary without assuming a fixed npm prefix:

```bash
NPM_ROOT="$(npm root -g)"
CODEX_BIN_DIR="$NPM_ROOT/@openai/codex/node_modules/@openai/codex-darwin-arm64/vendor/aarch64-apple-darwin/bin"
CODEX_NATIVE="$CODEX_BIN_DIR/codex"
CODEX_CODE_MODE_HOST="$CODEX_BIN_DIR/codex-code-mode-host"

"$CODEX_NATIVE" --version
ls -l "$CODEX_CODE_MODE_HOST"
```

It should report Codex 0.148.0.

### 3. Copy 0.148.0 and its code-mode host into Xcode's runtime directory

```bash
mkdir -p "$BASE/codex/0.148.0"

cp "$CODEX_NATIVE" "$BASE/codex/0.148.0/codex"
cp "$CODEX_CODE_MODE_HOST" "$BASE/codex/0.148.0/codex-code-mode-host"
chmod +x "$BASE/codex/0.148.0/codex" "$BASE/codex/0.148.0/codex-code-mode-host"

ls -la "$BASE/codex/0.148.0"
"$BASE/codex/0.148.0/codex" --version
```

Do not replace Xcode's original runtime directory. Keeping versions separately makes rollback simple.

**Important:** Codex 0.148.0 expects `codex-code-mode-host` to be present alongside `codex`. Copying only the main `codex` executable can make the model itself appear to work while Xcode-native tool calls fail. In the tested setup, Xcode's `xcode-tools` MCP server initialized successfully, but tool calls failed with an error like:

```text
failed to spawn code-mode host .../codex/codex-code-mode-host: No such file or directory (os error 2)
```

Installing the matching 0.148.0 `codex-code-mode-host` next to the 0.148.0 `codex` executable fixed the Xcode tool path.

### 4. Point Xcode at Codex 0.148.0

Fully quit Xcode first.

```bash
ln -sfn   "$BASE/codex/0.148.0"   "$BASE/XcodeVersions/$XCODE_BUILD/codex"
```

Verify:

```bash
readlink "$BASE/XcodeVersions/$XCODE_BUILD/codex"
"$BASE/XcodeVersions/$XCODE_BUILD/codex/codex" --version
```

The second command should report Codex 0.148.0.

### 5. Regenerate Xcode's model cache

Xcode stores the Codex model catalog at:

```text
~/Library/Developer/Xcode/CodingAssistant/codex/models_cache.json
```

Move the old cache out of the way:

```bash
CODEX_HOME="$HOME/Library/Developer/Xcode/CodingAssistant/codex"

mv   "$CODEX_HOME/models_cache.json"   "$CODEX_HOME/models_cache.json.backup"   2>/dev/null || true
```

Launch Xcode and open its Codex integration. Codex should fetch a fresh model catalog using the newer runtime.

Check for Luna:

```bash
grep -n '"gpt-5.6-luna"' "$CODEX_HOME/models_cache.json"
```

The regenerated cache in the tested setup identified the client as 0.148.0 and contained GPT-5.6 Luna, including its supported reasoning levels.

### 6. Configure Luna

Xcode's Codex configuration is:

```text
~/Library/Developer/Xcode/CodingAssistant/codex/config.toml
```

Use:

```toml
model = "gpt-5.6-luna"
model_reasoning_effort = "xhigh"
```

Do **not** add a `service_tier` line.

Fully quit and restart Xcode after changing the configuration.

The resulting tested configuration is:

```text
GPT-5.6 Luna
reasoning: xhigh
service tier: standard/default
authentication: ChatGPT account
```


## Verify Xcode-native tools

After restarting Xcode, verify that Codex is using Xcode's native tools rather than guessing from project context. With a project open, use this prompt:

```text
Use the Xcode tools to tell me the currently open project's name and list its top-level project files. Do not use shell commands or direct filesystem inspection.
```

In the tested setup, Codex reported using Xcode's `XcodeLS` tool and returned the live project navigator contents.

For a stronger end-to-end test, ask Codex to make an obvious source edit through Xcode:

```text
Using only the Xcode tools, edit ContentView.swift and change the displayed text in the default SwiftUI view to "Xcode Tools Are Working". Do not use shell commands or direct filesystem access. After making the change, tell me which Xcode tool you used.
```

The source edit was successfully applied in the tested 0.148.0 configuration. This confirms that the Xcode-native tool bridge works when both matching runtime binaries are installed.

## Using GPT-5.6 Sol

Change `config.toml`:

```toml
model = "gpt-5.6-sol"
model_reasoning_effort = "medium"
```

Restart Xcode.

`low` and `medium` were successfully tested with Sol through Xcode 26.6 and Codex 0.148.0.

There is a public Codex 0.148.0 bug report involving `prompt_cache_retention` being sent to GPT-5.6 Sol on some configurations. If Sol requests fail unexpectedly with 0.148.0, Codex 0.147.0 is a useful fallback.

## Verify what Xcode is actually using

Xcode's Codex logs are stored at:

```text
~/Library/Developer/Xcode/CodingAssistant/codex/logs_2.sqlite
```

A successful request in the tested setup logged:

```text
model=gpt-5.6-luna
codex.turn.reasoning_effort=xhigh
auth_mode=Some(Chatgpt)
```

Query recent Luna requests with:

```bash
DB="$HOME/Library/Developer/Xcode/CodingAssistant/codex/logs_2.sqlite"

sqlite3 "$DB" "
SELECT feedback_log_body
FROM logs
WHERE feedback_log_body LIKE '%model=gpt-5.6-luna%'
ORDER BY ts DESC, ts_nanos DESC, id DESC
LIMIT 5;
"
```

This verifies that Xcode is actually honoring the configured model and reasoning level.

## macOS permission prompt

The first time Xcode launches a newly copied Codex executable, macOS may ask whether it should be allowed to access the Documents folder.

This is expected. macOS privacy controls can treat a new executable/version/path as a new requester even though Xcode launches it.

If your Xcode projects are in Documents and you want Codex to work with them, allow the access.


## Codex 0.154.0 and GPT-6 Astra testing

Codex 0.154.0 was also tested because newer Codex releases add support for GPT-6 Astra. The release package contained both required Apple Silicon runtime binaries:

```text
codex
codex-code-mode-host
```

Both were copied into a separate `0.154.0` Xcode runtime directory and the Xcode symlink was pointed at it. This avoided changing or deleting the known-good 0.148.0 installation.

With Xcode 26.6, however, the Codex **Account** row remained on an indefinite spinner and account initialization never completed. This was the same general failure previously seen with `0.154.0-alpha.6.2`. Because the Xcode/Codex account handshake failed before normal operation, GPT-6 Astra could not be tested through Xcode's built-in Codex integration.

This means 0.154.0 should not currently replace 0.148.0 for this Xcode 26.6 configuration. It does **not** establish that Astra itself is incompatible with Xcode. The failure occurs earlier, at the Xcode/app-server account initialization layer.

If experimenting with future Codex releases, keep 0.148.0 intact and install each candidate in its own directory. Switching back is then just a symlink change followed by a full Xcode restart.

## Rollback

If you already copied Codex 0.147.0 into Xcode's agent directory:

```bash
BASE="$HOME/Library/Developer/Xcode/CodingAssistant/Agents"
XCODE_BUILD="17F113"

ln -sfn   "$BASE/codex/0.147.0"   "$BASE/XcodeVersions/$XCODE_BUILD/codex"
```

To restore Xcode's original runtime, use the exact target saved before making changes:

```bash
ln -sfn   "$ORIGINAL_CODEX_TARGET"   "$BASE/XcodeVersions/$XCODE_BUILD/codex"
```

Fully quit and restart Xcode after changing runtimes. You may also want to regenerate `models_cache.json`.

## Xcode updates

The runtime path contains the Xcode build number:

```text
~/Library/Developer/Xcode/CodingAssistant/Agents/XcodeVersions/17F113/codex
```

A future Xcode update may create a new directory under `XcodeVersions`.

Check with:

```bash
ls -la "$HOME/Library/Developer/Xcode/CodingAssistant/Agents/XcodeVersions"
```

If the modification stops working after an Xcode update, inspect the new build directory and repeat the symlink step if that Xcode build remains compatible with Codex 0.148.0.

Do not assume newer Codex releases will work just because 0.148.0 does. In this testing, 0.149 failed through Xcode 26.6, and both 0.154.0-alpha.6.2 and the 0.154.0 release failed during Xcode's Codex account initialization. The 0.154.0 release was tested with both `codex` and its matching `codex-code-mode-host`, so the account failure was not caused by the missing-helper problem that affected the initial 0.148.0 tool test.

## Recommended configuration

For routine Xcode coding:

```toml
model = "gpt-5.6-luna"
model_reasoning_effort = "xhigh"
```

No `service_tier` line.

Use Sol selectively when its additional capability is worth the higher ChatGPT subscription usage.

## References

- [OpenAI Codex repository](https://github.com/openai/codex)
- [Codex 0.148.0 release](https://github.com/openai/codex/releases/tag/rust-v0.148.0)
- [Codex 0.148.0 GPT-5.6 Sol `prompt_cache_retention` issue](https://github.com/openai/codex/issues/39397)
- [Codex releases](https://github.com/openai/codex/releases)
