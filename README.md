# Pulse design system — agent skill

A portable skill that teaches an AI agent how to design screens in Figma with the
Pulse / CV-Bot design system, written from a full build of the Kravmatrise
screens and an audit of the component library.

## What's here

```
skills/pulse-figma-design/
├── SKILL.md                        the method: probe, build, measure, look
└── references/
    ├── preflight.md                probes to run before trusting a component
    ├── build-patterns.md           page shell, card, table and slot recipes
    ├── api-gotchas.md              Figma plugin API traps
    ├── audit.md                    verification scripts, and their blind spot
    └── pulse-inventory.md          tokens, component IDs, known-broken list

agents/
├── pulse-ui-designer.md            builds; asks before changing components
└── pulse-design-reviewer.md        reports only; never edits
```

The skill is the knowledge; an agent is a role and a set of limits. Agents load
the skill rather than restating it, so there is one source of truth and nothing
to keep in sync.

The two agents are deliberately asymmetric. The designer can write but must ask
before touching an existing component, and can never publish. The reviewer can
measure but never fixes — the moment a reviewer edits, nobody knows what the file
looked like when it was reviewed.

Accessibility and UX live in the reviewer rather than in agents of their own,
because most of both is not visible in a Figma file. Contrast, focus states,
target size and colour-as-sole-meaning are checkable; keyboard order, screen
reader announcement and ARIA semantics are not. Missing states, content that
breaks with real data and inconsistent patterns are checkable; whether the design
solves the right problem is not — that needs user goals and research the file does
not contain.

So the reviewer covers the checkable subset of each and is required to name what
it could not judge. A clean report is never mistaken for an accessible design or
a validated one. Split those out into their own agents when there is an input
they would have and the reviewer does not — written flows and user goals, or a
codebase to inspect.

The general rule: **split agents by permission, not by topic.** Designer and
reviewer differ in what they are allowed to do, which is a boundary you can see
from outside. Layout, UX and accessibility are topics — agents split that way
share permissions, overlap in scope, and leave gaps at the seams.

## Installing it into an agent

**Claude Code** — copy the folders into either location:

```bash
cp -r skills/pulse-figma-design ~/.claude/skills/          # all projects
cp -r skills/pulse-figma-design /path/to/project/.claude/skills/   # one project

cp -r agents/. ~/.claude/agents/                           # all projects
cp -r agents/. /path/to/project/.claude/agents/            # one project
```

They load on the next session. The agent decides when to use the skill from the
`description` field in `SKILL.md`, and Claude decides when to delegate to an
agent from the `description` in its frontmatter.

**Any other agent** — the files are plain Markdown with no runtime dependencies.
Paste `SKILL.md` into a system prompt and attach the reference files, or hand the
whole folder over as context. The JavaScript samples target the Figma plugin API
as exposed through the Figma MCP's `use_figma` tool.

## What it's actually for

The short version of the method: **a component library is a set of claims, not
guarantees.** In this file, components accept properties and silently ignore
them, expose slots that cannot grow, and define the same property three times
under three IDs. Several of these report success through the API.

So the skill pushes an agent toward probing components before designing around
them, measuring after every structural change, and screenshotting before
declaring anything finished. Every real defect found during the build was
invisible in the API response and obvious in the picture.

It also carries the lesson that cost the most to learn: **passing a token audit
is not the same as being right.** A screen can use a variable at every level and
still sit in a rhythm nothing else in the file uses. The fix is to read the
nearest finished screen first and mirror it — no script catches that one.

## Scope

The method transfers to any Figma design system. `pulse-inventory.md` is specific
to `CV-Bot – Designfiler` (file key `hrX50tyQXpSb7phWJ5y5Dp`) — swap it out for
another system, and re-verify it periodically, since the known-broken list is a
snapshot rather than a live view.
