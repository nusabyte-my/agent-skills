# Agent skills — Malaysia / WhatsApp / agent fleets

Three production-tested [Agent Skills](https://agentskills.io) for work that the public
skill registries don't cover. Written from real deployments, sanitized of all client
data, infrastructure addresses and credentials.

| Skill | What it does |
|---|---|
| [`myinvois-einvoice-readiness`](skills/myinvois-einvoice-readiness/SKILL.md) | Audit a ledger for LHDN MyInvois readiness, build UBL 2.1 drafts, never invent a TIN |
| [`whatsapp-cloud-api-gateway`](skills/whatsapp-cloud-api-gateway/SKILL.md) | Multi-tenant WhatsApp Business Cloud API gateway on FastAPI + Postgres RLS |
| [`agent-fleet-task-bus`](skills/agent-fleet-task-bus/SKILL.md) | Operate N agent containers on a filesystem task bus — and tell "working" from "green" |

## Install

```bash
# all three
npx skills add nusabyte-my/agent-skills

# one, without cloning
npx skills add nusabyte-my/agent-skills --skill whatsapp-cloud-api-gateway

# Claude Code
cp -r skills/* ~/.claude/skills/

# Hermes
cp -r skills/* ~/.hermes/skills/
```

Each skill is a plain folder with a `SKILL.md` carrying YAML frontmatter (`name`,
`description`). No runtime, no dependencies, no scripts.

## Why these three

A search across the 3,200-skill AgentSkillExchange index and the skills.sh registry
returns **zero** results for `myinvois`, `e-invoice`, `malaysia`, or `whatsapp` as a
skill topic. These are the gaps the ecosystem has because most skill authors aren't
building for Malaysian compliance or for fleets of agents instead of one.

They are also the three areas where the expensive mistakes are invisible at the time:

- An e-invoice readiness gate "passed" by inventing a TIN fails at LHDN weeks later.
- A WhatsApp gateway with app-level tenant scoping leaks one client's chats into
  another's panel with no error.
- An agent fleet where every container is healthy and the task queue grows forever.

Each skill documents the signal that *looks* like success and isn't.

## Conventions

- **Draft-only.** Where the skill touches money, compliance or a customer, the agent
  drafts and a human sends. No exceptions encoded as "convenience" flags.
- **Errors are loud.** A blocked gate exits non-zero so a scheduler can branch.
- **No invented compliance data.** A red gate naming the missing input is the correct
  output.
- **Sanitized.** No hostnames, IPs, tenant names, tokens or client data. Everything is
  described structurally.

## License

MIT. See [LICENSE](LICENSE).

Author: Shaidan Shaari ([@nusabyte-my](https://github.com/nusabyte-my)) — NusaByte IT Solutions.
