# Connecting Claude to Unity with MCP for Unity (Windows)

A step-by-step memo for letting Claude (Claude Desktop, and chats linked to it) control the Unity Editor through the **MCP for Unity** package — inspect scenes, create GameObjects, edit scripts, read the console, and more.

Tested with:

| Component | Version |
|---|---|
| Windows | 11 |
| MCP for Unity (Unity package + `mcpforunityserver`) | v10.3.0 |
| Claude Desktop | Microsoft Store build (Nov 2026) |
| Transport | `stdio` |

---

## How it fits together

```
Claude (chat)  ──►  Claude Desktop  ──starts──►  unityMCP server (uvx, stdio)  ──socket :6400──►  Unity Editor (MCP for Unity bridge)
```

- **Claude Desktop** launches the MCP server from its config file when it starts.
- The **MCP server** talks to the **Unity Editor** over a local socket (default port `6400`).
- Both sides must use the **same transport**. This guide uses `stdio`, which Claude Desktop launches and manages itself.

---

## Prerequisites

1. **Unity project** open in the Unity Editor.
2. **MCP for Unity** package installed in the project
   (see the package's install instructions — typically via Package Manager → *Add package from git URL*).
3. **uv / uvx** installed. The Unity window's **Deps** tab can check and install it.
   Default location: `C:\Users\<you>\.local\bin\uvx.exe`
4. **Claude Desktop** installed and signed in.

---

## Step 1 — Set the transport to `stdio` in Unity

1. In Unity: **Window → MCP for Unity**.
2. On the **Connect** tab, set **Transport** to **Stdio**.

> ⚠️ If the window shows *"Claude Desktop is configured for 'stdio' but server is set to 'HTTP Local'"* or **Transport Mismatch**, the two sides disagree. Setting Transport to **Stdio** fixes it.

---

## Step 2 — Find the config file Claude Desktop *actually* reads

This is the step that usually goes wrong.

MCP for Unity writes to the standard path:

```
C:\Users\<you>\AppData\Roaming\Claude\claude_desktop_config.json
```

**But if Claude Desktop was installed from the Microsoft Store**, Windows sandboxes the app and it reads its config from its package folder instead:

```
C:\Users\<you>\AppData\Local\Packages\Claude_<id>\LocalCache\Roaming\Claude\claude_desktop_config.json
```

(`<id>` is a random suffix, e.g. `Claude_pzs8sxrjxfjjc`.)

**How to tell:** if `...\Local\Packages\Claude_*` exists, you have the Store build — edit the file there. Easiest way to open the right file: **Claude Desktop → Settings → Developer → Edit Config**.

> 💡 Symptom of editing the wrong file: Unity says **Configured**, but Claude Desktop → Settings → Developer shows no `unityMCP` server.

---

## Step 3 — Add the `unityMCP` server to the config

Open the file in **Notepad** (Windows may associate `.json` with another app, e.g. After Effects).

The file already contains other settings (e.g. `"preferences"`). **Keep them** — only add an `mcpServers` block at the top level.

Before (end of file):

```json
  "coworkUserFilesPath": "C:\\Users\\<you>\\Claude"
}
```

After:

```json
  "coworkUserFilesPath": "C:\\Users\\<you>\\Claude",
  "mcpServers": {
    "unityMCP": {
      "command": "C:\\Users\\<you>\\.local\\bin\\uvx.exe",
      "args": [
        "--offline",
        "--from",
        "mcpforunityserver==10.3.0",
        "mcp-for-unity",
        "--transport",
        "stdio"
      ],
      "type": "stdio"
    }
  }
}
```

Notes:

- Note the **comma** added after the previous last line.
- Backslashes in paths must be doubled (`\\`).
- Match `mcpforunityserver==X.Y.Z` to your Unity package version. The exact block for your machine is shown in the Unity window under **Client Configuration → Manual Configuration** — copy it from there.
- If `mcpServers` already exists, add only the `"unityMCP": { ... }` entry inside it.
- Validate the file (e.g. paste it into a JSON validator) before saving.

---

## Step 4 — Fully restart Claude Desktop

Closing the window is **not** enough — it keeps running in the background.

1. Right-click the Claude icon in the **system tray** (bottom-right) → **Quit**.
2. Reopen Claude Desktop.

Check: **Settings → Developer** (本機 MCP 伺服器 / Local MCP servers) should list **unityMCP** with status **running** (執行中).

---

## Step 5 — Start the Unity session

Back in **Window → MCP for Unity → Connect**:

- The status should read **● Session Active (YourProject)** with a green dot.
- If it shows **No Session** (red), start it from that window, or close/reopen the window or the project.

> 💡 Symptom: Claude can see the Unity tools but calls fail with
> *"No Unity Editor instances found. Please ensure Unity is running with MCP for Unity bridge."*
> → the server is fine, but the Unity side has no active session. Fix Step 5.

---

## Step 6 — Test from Claude

Open a chat (start a **new** chat if the Unity tools don't appear in an existing one) and ask something like:

> "What scene is currently open in Unity?"

A working setup returns the active scene's name, path and root object count.

---

## Troubleshooting cheat sheet

| Symptom | Cause | Fix |
|---|---|---|
| Unity: **Transport Mismatch** (yellow) | Unity transport ≠ Claude Desktop config | Set Transport to **Stdio** (Step 1) |
| Unity says **Configured**, but Claude Desktop shows no `unityMCP` | Store build reads a different config file | Edit the file under `AppData\Local\Packages\Claude_*` (Step 2) |
| `unityMCP` not running / error in Settings → Developer | Bad JSON, or `uvx.exe` missing | Validate JSON; install uv via Unity **Deps** tab; check **View Logs** |
| Config edits ignored | Claude Desktop not fully restarted | Quit from system tray, reopen (Step 4) |
| *"No Unity Editor instances found"* | Unity bridge session not active | Start session in Unity (Step 5) |
| Tools missing in an old chat | Chat started before server was added | Start a new chat |

---

## Alternative: HTTP Local transport

MCP for Unity can also run as a local HTTP server (`http://127.0.0.1:8080`). In that mode you must click **Start Server** in Unity and use **Configure** so the client config matches. For Claude Desktop, `stdio` is simpler because Claude Desktop starts and stops the server itself.
