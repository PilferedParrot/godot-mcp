# Troubleshooting

Connection problems, common fixes, and how to verify each link in the chain.

## The 60-second checklist

Most "it doesn't work" reports come down to one of these:

1. **Is the Godot editor open?** The server connects to an addon running *inside the editor*. No editor, no connection.
2. **Is the addon enabled?** Check **Project > Project Settings > Plugins > Godot MCP**. Installing the addon doesn't enable it.
3. **What does the MCP panel say?** The **MCP** tab in the editor's bottom panel shows connection status: green means a client is connected, orange means the addon is listening but no client has connected yet.
4. **Did you restart your AI assistant after installing?** MCP clients typically launch servers at startup. A client started before the config change won't see the server.
5. **Do the ports match?** The addon listens on `6550` by default. If you enabled **Port override** in the MCP panel, the server needs a matching `GODOT_PORT` (see [Environment Variables](../INSTALL.md#environment-variables)).

## One client at a time

A Godot editor bridge serves a **single** godot-mcp connection. This matters once subagents or multiple terminals inherit the same MCP config.

- If a second godot-mcp client connects while the first is active, it is **rejected** (WebSocket close code `4001`) rather than displacing the original — your session keeps working.
- The rejected client retries automatically with backoff and connects as soon as the first client disconnects.
- A client that crashes without closing its socket doesn't block anyone for long: the addon considers a connection stale after **45 seconds** of silence (live clients heartbeat every 30 seconds) and lets the next client take over.

So if a tool call reports that another client holds the connection: close the other session, or just wait — the takeover is automatic.

## Port conflicts and overrides

If port `6550` is taken on your machine (or you run multiple Godot projects side by side):

1. In the editor's **MCP** bottom panel, enable **Port override**, pick a port, and click **Apply**.
2. Set `GODOT_PORT` to the same value in your MCP client config (an `env` block alongside `command`/`args`, or however your client passes environment variables).

The panel persists its settings to `project.godot`, so the override sticks per-project.

## WSL

Running the MCP server inside WSL2 while Godot runs on Windows works, but localhost won't cross that boundary by itself:

- In the MCP panel, set **Bind mode: WSL** so the addon listens on the Windows `vEthernet (WSL)` interface instead of `127.0.0.1`.
- The server auto-detects WSL and discovers the Windows host IP on its own; set `GODOT_HOST` only if discovery fails.

Full details in the [Installation Guide](../INSTALL.md#wsl-support).

### Tools are present but every call reports it can't reach Godot

This is the bind-mode boundary above, caught at call time rather than startup. The tools are registered regardless of whether Godot is reachable (so they always show up in your client), and calling one while disconnected returns a diagnostic — in WSL it names the fix directly: set **Bind mode: WSL** in the Godot MCP panel, or set `GODOT_HOST`. The server reconnects on its own once the bind mode is right; you don't need to restart it, and a tool call that failed a moment ago will succeed once the bridge is up.

If your client started the server while Godot was still in **Localhost** bind mode, that's fine now — the tools are there either way. (Older versions could end up with an empty tool list in this situation; if you're on one, the recovery is your client's MCP reconnect, e.g. `/mcp` in Claude Code, which re-fetches the tool list.)

## Server and addon versions disagree

The MCP panel shows both the addon version and (when connected) the server version, and warns when they differ. The two are released together and expected to match. To update the addon in your project:

```bash
npx @satelliteoflove/godot-mcp --install-addon /path/to/your/godot/project
```

Add `--force` if you intentionally need to downgrade. Restart the Godot editor afterwards.

## Where errors show up

Two different processes produce errors, and they surface in two different places:

- **Editor-side errors** — `@tool` script failures, import errors, addon errors, failures from scene/resource edits — come from `godot_editor_read` with the `get_log_messages` action. This is the "did my change break the editor?" channel.
- **Running game console output** (including `print()` and stderr) is intentionally not duplicated here — [minimal-godot-mcp](https://github.com/ryanmazzolini/minimal-godot-mcp) provides it over DAP. Runtime *state* (positions, velocities, custom data) is `godot_runtime_state` territory.

## 3D geometry is missing, but nothing errors

Check placement in the **running game** before changing lights or materials. For runtime-spawned meshes, use [godot_exec](tools/exec.md) `run` to inspect the live nodes. Compare their measured bounds and positions with where you expected them to be:

1. **Under the floor?** For each `MeshInstance3D`, `mesh_instance.global_transform * mesh_instance.get_aabb()` gives a world-space AABB. Compare its bottom and top Y with the actual supporting surface. Don't assume the floor is at `y = 0`: a `GridMap` cell's center is not its mesh's top surface. A downward ray from the bounds' center can find a collider; if it misses, try upward to look for a slab above the prop. This needs floor collision; without it, compare the surface's mesh bounds directly.
2. **At an unintended origin?** Compare `global_position` with the parent and siblings. Props clustered at world `x = 0, z = 0` may have read a global transform before entering the tree. An origin position is a clue, not proof of a bug.
3. **Outside the camera?** Use the active `Camera3D` for the relevant viewport. An AABB is outside the frustum if all eight corners are outside the same plane from `get_frustum()`. Testing only the node's origin can miss a mesh that extends into view. Off-frustum is normal for objects you aren't meant to see.

Also check the **game console** for `Condition "!is_inside_tree()"`. Set global coordinates only after the node has entered the tree:

```gdscript
# parent is already inside the scene tree
parent.add_child(prop)
prop.global_position = desired_world_position
```

**Correction to [#328](https://github.com/satelliteoflove/godot-mcp/issues/328):** on Godot 4.6.1, a plain `global_position = p` before `add_child()` stores `p` as the local position and resets rotation and scale; it doesn't necessarily put the node at the origin. Component writes such as `global_position.y = v` and read-modify-writes such as `global_position += v` first read zero outside the tree, losing the previous coordinates. See Godot's [global_position documentation](https://docs.godotengine.org/en/stable/classes/class_node3d.html#class-node3d-property-global-position).

The [MIT primitive-only reproduction](https://github.com/PilferedParrot/godot-mcp-328-invisible-geometry) includes a [runtime audit body](https://github.com/PilferedParrot/godot-mcp-328-invisible-geometry/blob/main/recipe/runtime_audit.body.txt) you can pass as `godot_exec run`'s `source`. Read its coverage notes: it needs colliders for surface checks, skips props with their own colliders, and checks only the root viewport's active camera. `CLEAN` means no findings among the nodes audited, not that everything is correctly placed or visible. [godot_validate_meshes](tools/validate-meshes.md) checks mesh data, not placement.

## CLI smoke test (paste-ready JSON-RPC)

To verify the server starts and can reach Godot without involving an MCP client, run it manually:

```bash
npx -y @satelliteoflove/godot-mcp
```

Then paste these stdio JSON-RPC frames, one per line:

**1) Initialize:**

```json
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"cli-test","version":"0"}}}
```

You should get a response naming the server and its version (the real response also carries `title`, `description`, `websiteUrl`, and an `instructions` string, trimmed here):

```json
{"result":{"protocolVersion":"2024-11-05","capabilities":{"tools":{}},"serverInfo":{"name":"godot-mcp","version":"<your installed version>","...":"..."}},"jsonrpc":"2.0","id":1}
```

**2) Call a tool** (requires the Godot editor open with the addon enabled):

```json
{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"godot_editor_read","arguments":{"action":"get_state"}}}
```

A healthy response carries the editor state:

```json
{"result":{"content":[{"type":"text","text":"{\n  \"current_scene\": null,\n  \"godot_version\": \"4.5.1-stable (official)\",\n  \"is_playing\": false,\n  \"main_screen\": \"unknown\",\n  \"open_scenes\": []\n}"}]},"jsonrpc":"2.0","id":2}
```

If step 1 works but step 2 times out, the server is fine and the problem is the WebSocket hop — work through the [checklist](#the-60-second-checklist) above. If you're using a port override, remember to start the server with matching env vars:

```bash
GODOT_HOST=... GODOT_PORT=... npx -y @satelliteoflove/godot-mcp
```

Other useful CLI flags: `--version`, `--help`.

## Still stuck?

Open a [GitHub issue](https://github.com/satelliteoflove/godot-mcp/issues) with your OS, Godot version, server version (`npx @satelliteoflove/godot-mcp --version`), and what the MCP panel shows. The more of the chain you've tested with the smoke test above, the faster we can pin it down.
