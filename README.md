# ai-skills

A portable collection of Claude Code skills, commands, and rules to copy into any project.

## Quick Start

Copy the `.claude/` directory into your project root:

```bash
cp -r .claude/ /path/to/your/project/.claude/
```

Cherry-pick individual skills or commands as needed — each is a standalone markdown file.

## Structure

```
.claude/
├── skills/           # 17 skills that guide AI behavior
│   ├── ask-questions-if-unspecified/
│   ├── designing-apis/
│   ├── documenting-systems/
│   ├── frontend-design/
│   ├── handling-errors/
│   ├── managing-state/
│   ├── organizing-files/
│   ├── planning-features/
│   ├── writing-app-readmes/
│   ├── writing-forms/
│   ├── writing-logs/
│   ├── writing-nextjs/
│   ├── writing-react/
│   ├── writing-react-query/
│   ├── writing-tests/
│   └── writing-typescript/
└── commands/         # Slash commands
    ├── create-pr.md
    ├── explain-system.md
    ├── investigate.md
    ├── pr-description.md
    ├── review-pr.md
    └── verify-branch.md
```

## Skills

| Category     | Skills                                                                                                         |
| ------------ | -------------------------------------------------------------------------------------------------------------- |
| **Planning** | `planning-features`, `ask-questions-if-unspecified`                                                            |
| **Frontend** | `writing-react`, `writing-nextjs`, `writing-forms`, `managing-state`, `writing-react-query`, `frontend-design` |
| **Backend**  | `designing-apis`, `writing-logs`                                                                               |
| **Language** | `writing-typescript`                                                                                           |
| **Testing**  | `writing-tests`                                                                                                |
| **Quality**  | `handling-errors`, `organizing-files`                                                                          |
| **Docs**     | `documenting-systems`, `writing-app-readmes`                                                                   |

## Commands

| Command            | Description                                                        |
| ------------------ | ------------------------------------------------------------------ |
| `/create-pr`       | Creates a GitHub PR with title, description, and branch target     |
| `/explain-system`  | Deep-dives a system or module: dependencies, data flow, gotchas    |
| `/investigate`     | Traces an error through the codebase to find root cause and fixes  |
| `/pr-description`  | Generates a structured PR description and writes it to a file      |
| `/review-pr`       | Reviews a PR: summarizes changes, flags issues, checks test gaps   |
| `/verify-branch`   | Verifies branch changes against design docs and tests features     |
