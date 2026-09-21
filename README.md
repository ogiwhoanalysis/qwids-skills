# QWIDS agent skills

Two skills that teach an AI agent to use [QWIDS 2.0][qwids] correctly: the
OECD's development-finance statistics (ODA, the CRS, the DAC tables).

They are plain Markdown with YAML frontmatter, the [Agent Skills][spec] format,
so any client that reads that format can load them.

See [EXAMPLES.md](EXAMPLES.md) for four worked cases, with real figures, where
an agent carrying these answers differently from one that is not.

| Skill | Use it when |
|---|---|
| [`qwids-development-finance`](qwids-development-finance/SKILL.md) | Getting a defensible figure out of the typed API, and reading the answer: routing, measures, prices, markers, refusals, citation. |
| [`qwids-sql-console`](qwids-sql-console/SKILL.md) | A question needs a join or a shape the typed API cannot express, so it goes through `POST /v1/sql`. |

## Why skills as well as an MCP server

QWIDS serves an MCP endpoint at `/mcp/` with four typed tools. That gives an
agent the ability to *call* QWIDS. It does not give it the judgement to call it
well: which of ten tables answers a question, why a refusal is correct rather
than an obstacle, that an empty policy marker means "not screened" rather than
zero, or that a figure from the SQL console has had none of the methodology
rules applied to it.

The tools are the hands. These are the briefing.

## Install

Copy a skill directory into wherever your client reads skills from:

```bash
git clone https://github.com/ogiwhoanalysis/qwids-skills.git
cp -r qwids-skills/qwids-development-finance ~/.claude/skills/
cp -r qwids-skills/qwids-sql-console ~/.claude/skills/
```

`~/.claude/skills/` is the user-wide location for Claude Code; a project-local
`.claude/skills/` works too, and other clients have their own. The `description`
line in each `SKILL.md` is what a client reads to decide whether a skill is
relevant, so it is written to be matched against a user's question rather than
to read well.

## Keeping them honest

Everything in these files is checkable against a live deployment. The column
counts, the limits, the tool names and the refusals were read off the running
service rather than written from memory. If QWIDS changes, `GET /v1/sql/schema`
and `GET /v1/routing` are the two endpoints that will say so first.

## Licence

The skills are documentation of a public API. The underlying statistics are
published by the OECD; cite the dataset and vintage that every QWIDS response
carries.

[qwids]: https://qwids-web.whitetree-87b148ac.francecentral.azurecontainerapps.io
[spec]: https://code.claude.com/docs/en/skills
