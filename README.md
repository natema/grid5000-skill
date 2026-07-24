# grid5000-skill

A portable **AI-agent skill** for the [Grid'5000](https://www.grid5000.fr) HPC testbed — the
knowledge an LLM coding agent needs to reserve nodes, submit jobs, and run GPU/ML workloads on
Grid'5000 (including the Inria **"abaca"** / production queue) *without* getting it wrong.

Grid'5000 uses the **OAR** scheduler (`oarsub`) — less widespread than SLURM, so agents rarely know
it well — together with a three-tier access → frontend → compute-node workflow and per-site,
un-backed-up storage. Out of the box, assistants tend to reach for the wrong commands or compute on
the frontend. This skill encodes the correct commands, the gotchas, and a ready-to-use end-to-end
ML workflow.

## What's in it

A single [`SKILL.md`](SKILL.md) — a self-contained knowledge document with YAML frontmatter
(`name` + `description`) following the [Agent Skills](https://github.com/anthropics/skills)
convention. It covers:

- **OAR** resource model and `oarsub` recipes (interactive, passive/batch, submission scripts,
  property filters, advance reservations)
- **GPU reservations** and CUDA on the default environment (no kadeploy needed for normal ML)
- The **abaca / production** queue: walltime tiers, cluster/GPU inventory, access requirements
- **Storage**: per-site homes, group NFS volumes, node-local scratch, and how to move data in/out
- **Python envs**: `module`, conda, and `uv` on nodes; outbound internet
- **kadeploy** (when you actually need custom OS / root)
- **Project isolation** for shared logins, and a list of common **gotchas**

Concrete examples use public information from the
[Grid'5000 wiki](https://www.grid5000.fr/w/) (site names, queue names, cluster/GPU models). No
personal account details are included — replace `<login>`, `<group>`, `<site>`, and storage hosts
with your own.

## Install

The skill is harness-agnostic prose; drop it wherever your agent loads skills from.

**Claude Code** (user-level, available in every project):
```bash
mkdir -p ~/.claude/skills/grid5000
cp SKILL.md ~/.claude/skills/grid5000/SKILL.md
```

**Codex CLI**:
```bash
mkdir -p ~/.codex/skills/grid5000
cp SKILL.md ~/.codex/skills/grid5000/SKILL.md
```

**Other agents / manual use:** paste `SKILL.md` into your system prompt, project instructions, or
RAG knowledge base. The frontmatter `description` is written to help an agent decide *when* the
skill is relevant; the body is the actual guidance.

## Scope & accuracy

Grid'5000's hardware inventory and per-user quotas change over time. The skill deliberately tells
the agent to **verify live** (`oarnodes`, `quota -s`, the site Hardware wiki page) rather than trust
hardcoded values. Cluster names (`grat`, `musa`, …) and GPU models are illustrative snapshots, not
guarantees.

## Contributing

Corrections and additional recipes welcome — especially for sites/clusters not covered here. Keep
`SKILL.md` harness-agnostic and free of personal account details (see `CLAUDE.md`).

## License

[MIT](LICENSE).
