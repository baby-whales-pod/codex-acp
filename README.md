# Codex ACP in an SBX sandbox

This project runs Codex and its ACP server in a Docker Sandboxes sandbox.
Zed, AionUI, or any other ACP client remains on the host and starts the stdio
server with `sbx exec -i`. No port is published: ACP uses JSON-RPC over stdin
and stdout.

## Contents

```text
codex-acp/
├── sbxenv.yaml
├── bin/codex-acp-sbx
├── configs/zed-settings.jsonc
├── kits/codex-acp/codex-acp.yaml
└── workspace/
```

`sbxenv.yaml` combines the pinned official Codex kit with the local SBX v3
mixin. The mixin installs `@agentclientprotocol/codex-acp@2.3.0`, which
includes a compatible version of the Codex CLI. The host wrapper keeps stdin
open with `-i` and does not allocate a pseudo-terminal, because a TTY would
corrupt the ACP JSON-RPC stream.

## Prerequisites

- Docker Desktop with Docker Sandboxes and the `sbx` command;
- a valid `sbx login` session;
- Zed or AionUI installed on the host;
- a compatible ChatGPT subscription or an OpenAI API key.

The following commands assume you are in this directory:

```sh
cd docker-demos/codex-acp
```

## 1. Authenticate with OpenAI

The recommended method when using a ChatGPT subscription is SBX OAuth
storage:

```sh
sbx secret set openai --oauth
```

For an API key, let SBX prompt for it so it is not written to your shell
history:

```sh
sbx secret set openai
```

The secret remains in the SBX credential store. The Codex kit provides the
process with the expected proxy-managed environment variable; the real value
is not copied into this repository. `NO_BROWSER=1` deliberately hides browser
login from inside the sandbox because authentication takes place on the host.

## 2. Create the environment

```sh
sbx env create
```

The command displays its plan and asks for confirmation. For a non-interactive
run:

```sh
sbx env create --force
```

Then verify the binaries:

```sh
sbx exec codex-acp npm list --global --depth=0 @agentclientprotocol/codex-acp@2.3.0
sbx exec codex-acp command -v codex-acp
sbx exec codex-acp codex --version
```

## 3. Verify the ACP transport

The ACP server waits for a client on stdin, so it is normal for this command
to remain open without displaying an interface:

```sh
./bin/codex-acp-sbx
```

Stop this test with `Ctrl-C`. To diagnose the sandbox:

```sh
sbx exec codex-acp sh -lc 'id; pwd; command -v codex-acp; command -v codex'
sbx policy log
sbx exec codex-acp sh -lc 'ls -la /home/agent/.codex/logs'
```

## Zed

Zed must start `codex-acp` in the sandbox with `sbx exec`, rather than use its
local Codex installation from the ACP registry.

1. Get the absolute workspace path:

   ```sh
   realpath ./workspace
   ```

2. In Zed, run `agent: open settings`, open **External Agents**, then choose
   **Add Agent > Add Custom Agent**.
3. Copy the entry from `configs/zed-settings.jsonc` into Zed's `settings.json`.
   Replace the `/ABSOLUTE/PATH/TO/...` path with the result from the previous
   step.
4. Open the `workspace` directory in Zed.
5. In the Agent Panel, select **Codex SBX** and create a thread.

Complete configuration:

```json
{
  "agent_servers": {
    "Codex SBX": {
      "type": "custom",
      "command": "sbx",
      "args": [
        "exec",
        "-i",
        "-w",
        "/absolute/path/to/codex-acp/workspace",
        "codex-acp",
        "codex-acp"
      ],
      "env": {}
    }
  }
}
```

The command corresponds exactly to:

```sh
sbx exec -i -w /absolute/path/to/codex-acp/workspace codex-acp codex-acp
```

The first `codex-acp` is the sandbox name; the second is the command executed
inside it. `-i` keeps stdin open for ACP. Do not add `-t`: a pseudo-terminal
would interfere with the JSON-RPC stream.

The `bin/codex-acp-sbx` wrapper remains available if you prefer to centralize
the sandbox name and workspace:

```json
{
  "agent_servers": {
    "Codex SBX (wrapper)": {
      "type": "custom",
      "command": "/absolute/path/to/codex-acp/bin/codex-acp-sbx",
      "args": [],
      "env": {
        "CODEX_ACP_SANDBOX": "codex-acp",
        "CODEX_ACP_WORKDIR": "/absolute/path/to/codex-acp/workspace"
      }
    }
  }
}
```

If it fails, run `dev: open acp logs` from the Zed command palette. If Zed was
started graphically and cannot find `sbx`, use its absolute path as the
`command` value or use the wrapper.

## AionUI

AionUI can create an ACP **Custom Agent** that uses a stdio process.

1. Open **Settings > Agents**, then the **Custom Agents** tab.
2. Create an agent and enter:

   - Name: `Codex SBX`
   - Connection/transport: `stdio` or `CLI`
   - Command/CLI path: the absolute path returned by
     `realpath ./bin/codex-acp-sbx`
   - Arguments: none
   - Environment: `CODEX_ACP_SANDBOX=codex-acp` and
     `CODEX_ACP_WORKDIR=<absolute path returned by realpath ./workspace>`

3. Use **Test Connection**, save, and enable the agent.
4. Create a conversation with **Codex SBX** and select `workspace` as the
   workspace.

Depending on the AionUI version, these fields may appear in an advanced JSON
editor. The contract remains the same: executable = wrapper, arguments = empty
array, environment = the two variables above, transport = stdio.

## Obsidian with the Agent Client plugin

The `sbxenv.obsidian.yaml` file lets you use an Obsidian vault as the sandbox
workspace. Obsidian and the plugin remain on the host; Agent Client starts the
wrapper, which launches `codex-acp` in the sandbox over stdio.

1. In `sbxenv.obsidian.yaml`, replace the `workspace:` value with the absolute
   path to your vault. The path must point to the vault root, not only to its
   `.obsidian` directory:

   ```yaml
   workspace: /absolute/path/to/my-vault
   ```

2. After configuring the OpenAI secret as described above, create the
   dedicated environment:

   ```sh
   sbx env create sbxenv.obsidian.yaml
   ```

   This configuration creates a sandbox named `codex-acp-obsidian`.

3. In Obsidian, open **Settings > Community Plugins > Browse**, search for
   **Agent Client**, then install and enable the plugin.
4. Open **Settings > Agent Client**, scroll to **Custom Agents**, choose
   **Add custom agent**, and enter:

   - Agent ID: `codex-sbx`
   - Display name: `Codex SBX`
   - Path: the absolute path returned by
     `realpath ./bin/codex-acp-sbx`
   - Arguments: none
   - Environment variables, one per line:

     ```text
     CODEX_ACP_SANDBOX=codex-acp-obsidian
     CODEX_ACP_WORKDIR=/absolute/path/to/my-vault
     ```

5. Click the robot icon in the Obsidian ribbon, select **Codex SBX** from the
   agent list, and start a conversation. You can mention notes with `@`; Codex
   can read and edit files in the vault mounted as its workspace.

The plugin must use the custom agent above rather than its local Codex preset.
Authentication remains managed by the SBX secret, so do not enter an API key
in Agent Client. Keep permission confirmation enabled and review proposed
changes before accepting them.

## Use another project

Update both of the following:

1. `workspace:` in `sbxenv.yaml`;
2. `CODEX_ACP_WORKDIR` in Zed or AionUI.

Then recreate the environment:

```sh
sbx env rm
sbx env create
```

You can also keep this file and replace the contents of `workspace/`.

## Lifecycle

```sh
sbx env create              # create
sbx exec -it codex-acp sh   # diagnostic shell only
sbx env rm                  # remove the sandbox
```

Never add `-t` when starting ACP. Never put a key in `sbxenv.yaml`, the
Zed/AionUI configuration, or the wrapper.

## References

- [Agent Client Protocol](https://agentclientprotocol.com/get-started/introduction)
- [Codex ACP](https://github.com/agentclientprotocol/codex-acp)
- [Zed external agents](https://zed.dev/docs/ai/external-agents)
- [Official Codex CLI documentation](https://developers.openai.com/codex/cli/)
- [AionUI](https://www.aionui.com/en)
- [Agent Client plugin for Obsidian](https://github.com/RAIT-09/obsidian-agent-client)
