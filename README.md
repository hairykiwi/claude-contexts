# claude-contexts

Claude Code context files (`CLAUDE.md`) for open source projects,
containing machine and project-specific build environment details.

## What is a CLAUDE.md?

[Claude Code](https://docs.anthropic.com/en/docs/claude-code) reads a
`CLAUDE.md` file from a project's root directory to load context at the
start of each session — active branch, build environment, known issues,
commit conventions, and any other details that help Claude give accurate,
project-specific assistance without needing to be re-briefed each time.

## Why a separate repo?

A `CLAUDE.md` file often contains machine-specific details (OS version,
hardware, local tool paths) that are useful for one developer's setup but
irrelevant or misleading to other contributors. Keeping these files in a
dedicated repo avoids polluting project history with personal environment
details, while still keeping them version-controlled and backed up.

## Usage

Clone this repo once:

```bash
git clone git@github.com:hairykiwi/claude-contexts.git ~/claude-contexts
```

Then symlink the relevant `CLAUDE.md` into your project root:

```bash
ln -s ~/claude-contexts/<project>/CLAUDE.md ~/<project>/CLAUDE.md
```

Add `CLAUDE.md` to the project's `.gitignore` so git ignores the symlink:

```bash
echo "CLAUDE.md" >> ~/<project>/.gitignore
```

Claude Code will then read the context file automatically at the start
of each session.

## Contents

| Project | Branch | Platform |
|---|---|---|
| [qelectrotech-source-mirror](https://github.com/hairykiwi/qelectrotech-source-mirror) | `qt6_cmake_joshua` | macOS Apple Silicon |

## Notes

- These files reflect one developer's local environment and are not
  official project documentation
- Build paths, tool versions, and known issues will drift over time —
  treat them as a live document, not a static reference
- Contributions or adaptations for other platforms are welcome via PR

## Licence

[CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/)