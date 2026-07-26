# 04. Terminal UI & User Interaction

Unlike traditional web or Electron chat interfaces, **Claude Code CLI** renders its entire interactive UI directly inside the terminal using **Ink**—a React renderer for command line interfaces.

---

## 🖥️ Terminal UI Architecture

```
┌───────────────────────────────────────────────────────────┐
│                      Node.js Terminal                     │
│               stdout / stdin (ANSI Escapes)               │
└───────────────────────────────────────────────────────────┘
                              ▲
                              │ (Renders DOM to Terminal)
┌───────────────────────────────────────────────────────────┐
│                      React-Ink Engine                     │
│ Location: [src/ink.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/ink.ts:17)                              │
│ Wrapper: `<ThemeProvider>`                                │
└───────────────────────────────────────────────────────────┘
                              ▲
                              │ (React Component Tree)
┌───────────────────────────────────────────────────────────┐
│                   Top-Level `<App />` Container            │
│ Location: [src/replLauncher.tsx](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/replLauncher.tsx:19)                │
└───────────────────────────────────────────────────────────┘
          │                                   │
          ▼                                   ▼
┌───────────────────┐               ┌───────────────────┐
│ Messages List     │               │ Interactive Input │
│ - User Prompts    │               │ - Text Box        │
│ - Assistant Text  │               │ - Auto-complete   │
│ - Tool Call Diffs │               │ - Slash Commands  │
└───────────────────┘               └───────────────────┘
```

---

## 🎨 How Ink Renders React to ANSI

[src/ink.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/ink.ts:17) initializes Ink's custom root:
1. **Theming**: Encloses all rendered nodes inside `<ThemeProvider>` ([src/components/design-system/ThemeProvider.js](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/components/design-system/ThemeProvider.js)), ensuring components like `ThemedBox` and `ThemedText` automatically respect active terminal color schemes.
2. **Flexbox Layout**: Ink uses Yoga layout under the hood to calculate flexbox positioning for terminal columns and rows.
3. **Differential Re-rendering**: Ink updates only changed ANSI lines in stdout, preventing screen flickering as text streams from the model.

---

## 💬 Key Interactive Components

Located in [src/components/](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/components/):

1. **Transcript & Message Rendering**:
   - Renders messages chronologically.
   - Renders markdown formatting, syntax-highlighted code blocks, and streamed response deltas.

2. **Tool Output & Diff Displays**:
   - Tool calls display animated spin indicators while running.
   - File edits show standard green/red `git diff` style additions and deletions.

3. **Interactive Permissions Dialog**:
   - When a tool requires user consent (e.g. running `rm -rf` in bash), Ink pauses rendering and presents an interactive selection box: `[Allow] [Deny]`.

4. **Footer & Cost Counter**:
   - Displays real-time token usage, session dollar cost, model name (e.g., `claude-3-7-sonnet`), and active context window percentage.

---

## ⌨️ Input Handling & Slash Commands

Input handling is powered by custom hooks and keyboard listeners:
- **Slash Commands**: Typing `/` triggers autocompletion for commands defined in [src/commands.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/commands.ts) (such as `/clear`, `/compact`, `/cost`, `/help`, `/review`).
- **Keybindings**: Configured in [src/keybindings/](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/keybindings/), handling multiline input toggle (`Option+Enter` / `Shift+Enter`), prompt navigation history (`Up`/`Down` arrow keys), and cancellation (`Ctrl+C`).

---

## 🔑 Key Code Locations

- **Ink Custom Renderer**: [src/ink.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/ink.ts:17)
- **REPL Launcher**: [src/replLauncher.tsx](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/replLauncher.tsx:12)
- **Commands Registry**: [src/commands.ts](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/commands.ts)
- **Design System & Theme**: [src/components/design-system/](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/components/design-system/)
- **Dialog Launchers**: [src/dialogLaunchers.tsx](/home/jerry/workspaces/personal/repos/cc-haha.worktrees/claude-code-cli-analysis-docs/src/dialogLaunchers.tsx)
