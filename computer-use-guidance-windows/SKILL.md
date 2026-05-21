---
name: computer-use-guidance-windows
version: 0.1.0
description: "Operation guidance for automating the Windows desktop via Computer Use: take screenshots, click / type / press shortcut keys / scroll / drag / wait, with best practices for the normalized coordinate system, multi-display awareness, auto-screenshot verification, zoom-for-detail, input method handling, and platform-specific recipes across apps, windows, files, system settings, terminal, office apps, and developer tools. Tool argument schemas are loaded lazily via qw_mcp_get when the runtime is in MCP lazy-load mode."
description_zh: "Windows 平台 Computer Use 桌面自动化操作指南：通过截图配合点击 / 文本输入 / 快捷键 / 滚动 / 拖拽 / 等待等 UI 操作完成桌面自动化，包含归一化坐标体系、多显示器感知、动作后自动截图验证、放大查看、输入法处理等最佳实践，覆盖应用、窗口、文件、系统设置、终端、办公、开发工具等常见场景的操作范式。MCP 懒加载模式下，工具的入参 schema 通过 qw_mcp_get 按需拉取。"
---

# Computer Use Guidance (Windows)

This skill provides Windows-specific operation guidance for desktop automation via Computer Use tools. It helps you perform common desktop tasks correctly and efficiently on Windows, covering app management, window control, file operations, system settings, terminal, office apps, and dev tools.

If your runtime exposes MCP tools through the lazy-load meta tools, read the **Lazy-load Bootstrap** section below first to obtain the authoritative tool schemas at runtime.

---

## Lazy-load Bootstrap (MCP 懒加载模式)

If your runtime exposes MCP tools through the lazy-load meta tools (`qw_mcp_list` / `qw_mcp_get` / `qw_mcp_call`) instead of listing every tool directly, **load the full Computer Use tool contract before executing any automation step**:

1. Call `qw_mcp_list({keyword: "computer-use"})` to enumerate all Computer Use tools exposed by this connector.
2. For every returned tool, call `qw_mcp_get({toolName})` to cache its input schema (argument names, required fields, enums).
3. Only after every Computer Use tool schema is loaded, start the task — do **not** guess argument shapes from this skill document alone.

This skill describes *when* to use which tool and *why*; it deliberately does not enumerate full input schemas. The authoritative schema always lives on the MCP server, reachable via `qw_mcp_get`. In direct (non-lazy) mode, schemas are already visible in the tool list and this bootstrap can be skipped.

---

## Tool Selection Strategy

Before resorting to Computer Use (GUI automation), always consider whether a more reliable and efficient tool is available. Follow this priority order:

| Priority | Tool Type | When to Use | Examples |
|----------|-----------|-------------|---------|
| 1st | **Built-in Tools** | File I/O, code editing, shell commands, git operations | `Read`, `Write`, `Edit`, `Bash`, `Glob`, `Grep` |
| 2nd | **MCP Tools** | Structured API interactions with specific services | Database queries, API calls, service-specific operations |
| 3rd | **Browser Tools** | Web page interaction with DOM-level precision | `browser-use` for navigating websites, filling forms, extracting web content |
| 4th | **Computer Use** | GUI-only operations with no programmatic alternative | System settings, native app UIs, drag-and-drop, visual verification |

**Key principles:**

- **Prefer programmatic over visual.** If a task can be done via CLI, API, or built-in tool, do NOT use Computer Use. For example, use a shell command to launch an app instead of clicking through the Taskbar.
- **Prefer Browser Tools over Computer Use for web tasks.** Browser tools operate at the DOM level with reliable element selection; Computer Use relies on screenshot-based coordinate guessing which is fragile for web pages.
- **Use Computer Use for what only it can do.** Native OS dialogs, system settings, desktop app UIs without automation APIs, multi-app workflows that span the entire desktop — these are where Computer Use shines.
- **Combine tools when appropriate.** A task might start with a shell command to open an app, then switch to Computer Use for GUI interactions within that app.
- **Launch GUI apps with non-blocking patterns.** Prefer `Start-Process <app>` (PowerShell) or `cmd /c start "" <app>` — a foreground launch can keep the tool call alive until the app exits. After launching, `wait` briefly and verify with a screenshot.

## General Best Practices

### 1. Screenshot First

Always take a screenshot (`screenshot`) before performing any operation to confirm the current screen state. Never guess coordinates or UI element positions. The returned image is pre-normalized (~1280px wide, aspect ratio preserved) and carries display metadata (`displayId`, normalized dimensions, current cursor position).

### 2. Operate-Wait-Verify Rhythm

Action tools (`click`, `mouse_move`, `drag`, `type`, `key`, `scroll`, `wait`) automatically return a screenshot in their result — you do **not** need a separate `screenshot` call after every action.

1. **Operate**: Execute the action.
2. **Wait**: The tool waits `screenshotDelay` ms (default 300ms) for the UI to settle.
3. **Verify**: Inspect the auto-returned screenshot.

For slow transitions (page loads, app launches), raise `screenshotDelay`, or use `wait({ms: ...})` which also returns a screenshot when done. When chaining rapid actions where intermediate frames are unnecessary, pass `returnScreenshot: false`.

### 3. Coordinate Accuracy

Because screenshots are pre-normalized, coordinate resolution is consistent regardless of the physical display resolution:

- Identify the target pixel `[x, y]` directly in the most recent screenshot and pass it together with `displayId` (from the screenshot's metadata) to `click` / `mouse_move` / `drag`.
- Always use coordinates from the **most recent** screenshot — never reuse old coordinates after the screen has changed.
- Aim for the **center** of the target element, not its edge.
- Do NOT attempt manual DPI / scale / offset conversion — the system handles it.
- When an element is too small to identify precisely, call `screenshot` with `zoomCenter: [x, y]` (and optional `zoomLevel` 2–6) for a magnified view — but click coordinates must still refer to the full (un-zoomed) screenshot.
- Screenshots include a red crosshair at the current cursor position by default. Use it to verify whether a click landed on the intended target; pass `hideCursor: true` only if the marker obscures visual inspection.

### 4. Multi-Display Awareness

When multiple displays are connected, always confirm which display you are operating on before clicking:

- Use `list_displays` to inventory displays (id / resolution / bounds / primary flag).
- In `screenshot` text output, check `【Current Display (Screenshot)】`, `【Coordinate System】` (normalized size + displayId to use), `【All Available Displays】`, and `【Current Cursor Position】`.
- Use `cursor_position` to get `{x, y, displayId}` directly when needed.
- For a full desktop overview in one step, call `screenshot({captureAllDisplays: true})` — returns one image per display.
- `drag` endpoints must be on the same `displayId`.

### 5. Input Method Awareness

Before typing text with `type`:

- Check if the correct input method is active (especially for CJK text).
- Use `Win+Space` or `Alt+Shift` to switch input methods.
- When typing URLs, commands, or code, ensure the input method is set to English/ASCII.

### 6. Error Recovery

When an operation doesn't produce the expected result (inspect the auto-returned screenshot first):

1. Press `Escape` to dismiss any unexpected dialogs or menus.
2. If the app is frozen, call `wait({ms: 3000})` — its auto-screenshot will show the current state.
3. If stuck, navigate back to a known state (click on desktop, open Start Menu via `Win`).

### 7. Scrolling Strategy

- Use `scroll` with moderate amounts (2–3 units) to avoid overshooting.
- The auto-returned screenshot shows the new viewport — check whether the target element is now visible.
- For long pages, scroll in increments rather than large jumps.

---

## Common Windows Operations

This section covers key steps for common desktop operations on Windows. Each operation describes the concept and critical steps — use screenshots to determine exact coordinates and UI element positions.

### 1. App Management

#### Non-Blocking Launch Rule
- When using shell commands to launch a Windows GUI app, do **not** rely on foreground launch forms that may keep the tool call alive until the app exits
- Avoid patterns like directly running a long-lived GUI process in the foreground; if a previous workflow used `start notepad` and it blocked, treat that pattern as unsafe for agent loops
- Prefer detached launch forms such as:
  - `powershell -NoProfile -Command "Start-Process notepad"`
  - `cmd /c start "" notepad`
- After launching, use `wait` for 500-1500ms, then take a screenshot to confirm the window is actually visible before interacting with it

#### Launch an App via Start Menu Search
- Press the `Win` key (or `Win+S`) to open Start Menu / Search
- Type the app name (e.g., "Chrome", "Visual Studio Code")
- Press `Enter` to launch the top result
- Wait 1-2 seconds for the app to open, then screenshot to confirm

#### Launch an App from Taskbar
- The Taskbar is typically at the bottom of the screen
- Screenshot to identify the app icon position in the Taskbar
- Click on the target app icon

#### Switch Between Apps
- Press `Alt+Tab` to show the task switcher overlay
- Continue holding `Alt` and press `Tab` to cycle through windows
- Release `Alt` to switch to the highlighted window
- Alternatively, click the app's icon in the Taskbar

#### Close an App
- Press `Alt+F4` to close the current foreground window/app
- If all windows of an app are closed, the app typically exits

#### Open Task Manager
- Press `Ctrl+Shift+Esc` to open Task Manager directly
- To force-end a process: find it in the list, right-click, select "End task"
- Alternatively: `Ctrl+Alt+Delete` then select "Task Manager"

### 2. Window Management

#### Minimize / Maximize / Close
- **Minimize**: `Win+Down` (if maximized, first press restores; second press minimizes)
- **Maximize**: `Win+Up`
- **Close**: `Alt+F4`
- The window control buttons (minimize/maximize/close) are at the top-right corner of windows

#### Snap Windows (Side by Side)
- **Snap left**: `Win+Left` — window fills the left half of the screen
- **Snap right**: `Win+Right` — window fills the right half
- **Snap to quadrant**: After snapping left/right, press `Win+Up` or `Win+Down` to snap to a quarter
- After snapping one window, Windows may suggest another window to fill the remaining space

#### Snap Layouts (Windows 11)
- Press `Win+Z` or hover over the maximize button to see Snap Layout options
- Select a layout zone to position the window
- Then select other windows to fill remaining zones

#### Task View
- Press `Win+Tab` to open Task View (shows all open windows and virtual desktops)
- Click on a window to bring it to focus
- Click on a desktop thumbnail at the top to switch desktops

#### Virtual Desktops
- **New desktop**: `Win+Ctrl+D`
- **Switch left/right**: `Win+Ctrl+Left` or `Win+Ctrl+Right`
- **Close current desktop**: `Win+Ctrl+F4`

#### Other Window Shortcuts
- **Show desktop**: `Win+D` (minimize all) — press again to restore
- **Minimize all except current**: `Win+Home`

### 3. File Operations

#### Open File Explorer
- Press `Win+E` to open File Explorer
- Or click the File Explorer icon in the Taskbar (folder icon)

#### Navigate to Common Locations
File Explorer's left sidebar contains Quick Access, This PC, and other locations:
- **Quick Access**: Shows frequently accessed folders and recent files
- **This PC**: Shows drives (C:, D:, etc.) and system folders
- **Desktop**: Usually accessible from Quick Access or `C:\Users\{username}\Desktop`
- **Downloads**: Usually accessible from Quick Access or `C:\Users\{username}\Downloads`
- **Navigate to specific path**: Click the address bar at the top (or press `Ctrl+L`), type the path, press `Enter`

#### File Operations
- **Copy**: Select file(s) then `Ctrl+C`
- **Paste**: Navigate to destination then `Ctrl+V`
- **Cut** (move): `Ctrl+X` then `Ctrl+V` at destination
- **Delete**: Select file(s) then `Delete` (moves to Recycle Bin) or `Shift+Delete` (permanent delete)
- **Rename**: Select file, press `F2`, type new name, press `Enter`
- **New Folder**: `Ctrl+Shift+N`
- **Select all**: `Ctrl+A`
- **Undo**: `Ctrl+Z`

#### Preview Panel
- Press `Alt+P` in File Explorer to toggle the Preview Panel on the right side

#### Search in File Explorer
- Press `Ctrl+E` or `Ctrl+F` to focus the search bar
- Type the search term and press `Enter`

### 4. System Settings

#### Open Settings
- Press `Win+I` to open Settings
- Or press `Win` key, type "Settings", press `Enter`

#### Navigate to Common Settings Modules
Once Settings is open, use the left sidebar or search bar:

| Setting | Navigation Path |
|---------|----------------|
| WiFi | Settings > Network & Internet > Wi-Fi |
| Bluetooth | Settings > Bluetooth & devices |
| Display | Settings > System > Display |
| Sound | Settings > System > Sound |
| Notifications | Settings > System > Notifications |
| Keyboard | Settings > Time & language > Typing |
| Mouse / Touchpad | Settings > Bluetooth & devices > Mouse / Touchpad |
| Network | Settings > Network & Internet |
| Privacy & Security | Settings > Privacy & security |
| Appearance (Theme) | Settings > Personalization |
| Apps & Features | Settings > Apps > Installed apps |
| Windows Update | Settings > Windows Update |

- Use the search field at the top of Settings to quickly find a setting
- Screenshot after navigating to confirm you're in the right section

### 5. Text Editing (Universal)

#### Basic Operations
- **Copy**: `Ctrl+C`
- **Paste**: `Ctrl+V`
- **Cut**: `Ctrl+X`
- **Undo**: `Ctrl+Z`
- **Redo**: `Ctrl+Y`
- **Select All**: `Ctrl+A`

#### Text Selection
- **Select character by character**: `Shift+Left/Right`
- **Select word by word**: `Ctrl+Shift+Left/Right`
- **Select to start of line**: `Shift+Home`
- **Select to end of line**: `Shift+End`
- **Select to start/end of document**: `Ctrl+Shift+Home/End`
- **Select a line**: `Home` then `Shift+Down` or `Shift+End`

#### Navigation
- **Jump to start/end of line**: `Home` / `End`
- **Jump word by word**: `Ctrl+Left/Right`
- **Jump to start/end of document**: `Ctrl+Home/End`

#### Find and Replace
- **Find**: `Ctrl+F`
- **Find and Replace**: `Ctrl+H`
- Type search term, use `Enter` or arrow buttons to navigate matches
- In Replace mode, type replacement text and click "Replace" or "Replace All"

### 6. Browser Operations

These shortcuts work in most browsers (Chrome, Edge, Firefox):

#### Tab Management
- **New tab**: `Ctrl+T`
- **Close current tab**: `Ctrl+W`
- **Reopen closed tab**: `Ctrl+Shift+T`
- **Switch to tab by number**: `Ctrl+1` through `Ctrl+9` (Ctrl+9 = last tab)
- **Next tab**: `Ctrl+Tab`
- **Previous tab**: `Ctrl+Shift+Tab`

#### Navigation
- **Focus address bar**: `Ctrl+L` or `F6` or `Alt+D`
- **Go back**: `Alt+Left`
- **Go forward**: `Alt+Right`
- **Refresh**: `F5` or `Ctrl+R`
- **Hard refresh** (bypass cache): `Ctrl+Shift+R` or `Ctrl+F5`

#### Other Useful Shortcuts
- **Open Developer Tools**: `F12` or `Ctrl+Shift+I`
- **View page source**: `Ctrl+U`
- **Zoom in**: `Ctrl+=`
- **Zoom out**: `Ctrl+-`
- **Reset zoom**: `Ctrl+0`
- **Full screen**: `F11`
- **Downloads**: `Ctrl+J`

### 7. Terminal / Command Line

#### Open Windows Terminal
- Press `Win` key, type "Terminal" or "Windows Terminal", press `Enter`
- This opens Windows Terminal (default on Windows 11)
- On older Windows: search for "PowerShell" or "cmd"
- If you need to launch another GUI app from Terminal for later Computer Use interaction, prefer `Start-Process` or `cmd /c start "" <app>` so the shell tool returns immediately instead of waiting for the GUI app to exit

#### Terminal Basics
- **New tab**: `Ctrl+Shift+T`
- **New tab with specific profile**: `Ctrl+Shift+1/2/3` (PowerShell, CMD, etc.)
- **Switch tabs**: `Ctrl+Tab` / `Ctrl+Shift+Tab`
- **Close tab**: `Ctrl+Shift+W`
- **Split pane**: `Alt+Shift+D` (duplicate) or `Alt+Shift+-/+` (horizontal/vertical)
- **Switch between panes**: `Alt+Arrow`
- **Clear screen**: Type `cls` + `Enter` (CMD) or `Clear-Host` / `cls` + `Enter` (PowerShell)

#### Command Control
- **Interrupt command**: `Ctrl+C` (stops the running command)
- **End of input**: `Ctrl+Z` then `Enter` (sends EOF on Windows)

#### PowerShell vs CMD
- **PowerShell**: Modern, supports `ls`, `cd`, `cat` (as aliases), pipeline with objects, script files `.ps1`
- **CMD**: Legacy, uses `dir`, `cd`, `type`, batch files `.bat`/`.cmd`
- Windows Terminal can run both — use the dropdown next to the `+` button to select

#### Text Editing in Terminal
- **Move cursor to start of line**: `Home`
- **Move cursor to end of line**: `End`
- **Search command history**: `Ctrl+R` (PowerShell with PSReadLine) or `F7` (CMD)
- **Previous command**: `Up Arrow`
- **Copy selected text**: `Ctrl+C` (when text is selected) or `Ctrl+Shift+C`
- **Paste**: `Ctrl+V` or `Ctrl+Shift+V` or right-click

### 8. Office Software

#### General Windows App Ribbon Pattern
Windows Office apps use the Ribbon interface:
1. Press `Alt` to activate Ribbon key tips (letters appear on tabs)
2. Press the corresponding letter to navigate to a tab (e.g., `Alt+H` for Home)
3. Press further letters to activate specific commands
4. Alternatively, click on Ribbon tabs and buttons directly

#### Microsoft Word
- **New document**: `Ctrl+N`
- **Open document**: `Ctrl+O`
- **Save**: `Ctrl+S`
- **Save As**: `F12` or `Ctrl+Shift+S`
- **Print**: `Ctrl+P`
- **Export to PDF**: File tab > Export > Create PDF/XPS Document
- **Bold/Italic/Underline**: `Ctrl+B` / `Ctrl+I` / `Ctrl+U`

#### Microsoft Excel
- **Navigate cells**: Arrow keys to move between cells
- **Edit cell**: Press `F2` or double-click on a cell to start editing
- **Confirm entry**: Press `Enter` (moves down) or `Tab` (moves right)
- **Cancel editing**: Press `Escape`
- **Select range**: Click a cell, hold `Shift`, click another cell; or click and drag
- **Select entire row**: Click the row number on the left
- **Select entire column**: Click the column letter at the top
- **Insert formula**: Type `=` followed by the formula (e.g., `=SUM(A1:A10)`)
- **Auto-sum**: `Alt+=` (inserts SUM formula for adjacent cells)
- **New worksheet**: Click `+` tab at the bottom, or `Shift+F11`

#### Microsoft PowerPoint
- **New slide**: `Ctrl+M`
- **Start slideshow from beginning**: `F5`
- **Start from current slide**: `Shift+F5`
- **Exit slideshow**: `Escape`
- **Duplicate slide**: `Ctrl+D` (with slide selected in panel)
- **Switch between Normal/Outline/Slide Sorter views**: View tab in Ribbon

### 9. Dev Tools

#### Visual Studio Code (VSCode)
- **Launch**: `Win` key, type "Visual Studio Code" or "code", press `Enter`
- **Command Palette**: `Ctrl+Shift+P` — the most powerful shortcut, search for any command
- **Quick Open file**: `Ctrl+P` — type filename to quickly open any file in the project
- **Toggle Sidebar**: `Ctrl+B`
- **Toggle integrated terminal**: `` Ctrl+` ``
- **New terminal**: `` Ctrl+Shift+` ``
- **Split editor**: `Ctrl+\`
- **Go to definition**: `F12` or `Ctrl+Click`
- **Find in files**: `Ctrl+Shift+F`
- **Find and replace in file**: `Ctrl+H`
- **Toggle line comment**: `Ctrl+/`
- **Move line up/down**: `Alt+Up/Down`
- **Multi-cursor**: `Alt+Click` to add cursors at multiple positions
- **Open Settings**: `Ctrl+,`
- **Open Extensions**: `Ctrl+Shift+X`

#### Visual Studio
- **Launch**: `Win` key, type "Visual Studio", press `Enter`
- **Build solution**: `Ctrl+Shift+B`
- **Run (Start Debugging)**: `F5`
- **Run without debugging**: `Ctrl+F5`
- **Stop debugging**: `Shift+F5`
- **Step over**: `F10`
- **Step into**: `F11`
- **Toggle breakpoint**: `F9`
- **Go to file**: `Ctrl+T` (Go To All)
- **Solution Explorer**: `Ctrl+Alt+L`
- **Error List**: `Ctrl+\, E` or View > Error List
- **Terminal/Console**: View > Terminal
- **Find in files**: `Ctrl+Shift+F`
- **Find and replace**: `Ctrl+Shift+H`
