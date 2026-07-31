# AGENTS.md

Editing conventions for this repository, for human and AI contributors alike.

## What this repo is

A single published **AI-agent skill** for the Grid'5000 HPC testbed. The entire deliverable is
[`SKILL.md`](SKILL.md) — a self-contained knowledge document (YAML frontmatter + Markdown body) that
teaches an LLM agent how to use Grid'5000 (OAR scheduler, GPU reservations, abaca/production queue,
storage, Python envs). There is no build, no test suite, and no application code. Editing means
editing prose.

## Non-negotiable constraints

1. **`SKILL.md` must stay harness-agnostic.** It is installed into Claude Code, Codex, and other
   agents (see the README install section), and pasted into system prompts / RAG stores. Do **not**
   introduce assistant-specific or tool-specific instructions into `SKILL.md`. Keep it about
   *Grid'5000*, not about any one agent runtime — in particular, never refer to "the user's global
   preferences" or anything else that only exists in one harness's configuration.

2. **Never commit personal account details.** This repo is a genericized version of a private skill.
   Personal identifiers were deliberately stripped and must not return: real logins (use `<login>`),
   team/Unix group names and their `/srv/storage/<group>@<host>` paths (use placeholders), specific
   quota numbers raised for one account, and dated "verified on <date>" notes tied to one user.
   Concrete **public** Grid'5000 facts from the [wiki](https://www.grid5000.fr/w/) are fine and
   encouraged as examples (site names, queue names `production`/`abaca`, cluster names like `grat`/
   `musa`, GPU models). The line is: *public infrastructure fact = OK; private account fact = not OK.*

3. **Prefer "verify live" over hardcoded inventory.** Grid'5000 hardware and quotas rotate. When
   adding cluster/GPU/quota specifics, pair them with the command that re-checks them (`oarnodes`,
   `quota -s`, the site Hardware wiki page) so the skill degrades gracefully as the testbed changes.

4. **Don't turn observed behaviour into a stated quota.** What one account or one site sees is not a
   testbed-wide rule. Document limits as "check yours with `<command>`", plus fair-use etiquette on
   shared resources.

5. **Cite the source.** When adding a fact from the Grid'5000 wiki, reference the page; when adding
   one confirmed on the testbed, say which command confirmed it.

## Frontmatter format

The top-of-file YAML is the Agent Skills convention: a `name` (kebab-case, matching the install
directory `grid5000/`) and a `description` written as *when to invoke* guidance for an agent's skill
router. If you rewrite the body, keep the `description` an accurate trigger summary — it is how an
agent decides the skill is relevant.

## Relationship to the private source

This public `SKILL.md` was derived from a private, user-specific version kept in the author's local
skills directory (which retains their login, group, and storage specifics). The two can drift; when
syncing improvements *from* the private version, re-apply the genericization in constraint #2. Do
not copy the private version verbatim.

Harness-specific instruction files (e.g. `CLAUDE.md`) are deliberately **not** committed — they are
gitignored so this repo stays vendor-neutral. Put anything worth sharing here instead.

## Sibling repos

Same pattern, same conventions — check them for consistency before changing structure here:
[`azzurra-skill`](https://github.com/natema/azzurra-skill) and
[`jeanzay-skill`](https://github.com/natema/jeanzay-skill).
