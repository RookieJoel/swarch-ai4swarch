**Language:** [ภาษาไทย](00-prompt-log.md) | English

# Prompt log

**Name / Student ID:** Suttanop Chanah 6632239821
**Agent(s) used:** Claude Code (CLI)
**Model(s):** Claude Opus 5.5 (`claude-opus-5-5`)
**Plan / tier:** Pro

Log every request you send to the agent. "Files the agent read" is what the agent reported, or what you saw it open.

| # | Task / run | Agent + model | Message sent (short) | Files the agent read | Stayed within the allowed evidence? | Useful? (1–5) | Notes |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code + Opus 5.5 | `prompts/1-recover-architecture.md` run A, evidence `protos/demo.proto` (sent over several messages: summarise task → read AGENTS.md → run A) | `prompts/1-recover-architecture.md`, `AGENTS.md`, `system/online-boutique/protos/demo.proto`, `exercise/01-recover-architecture.en.md` (as reported in `outputs/1A-architecture.md` §2) | Mostly. It opened the worksheet to look up the missing `evidence` value. **Check:** you also asked it to "look through the project and find the related evidence". If it opened more files, Run A is no longer interfaces-only. | (you fill) | Output: `exercise/outputs/1A-architecture.md`. `evidence` wasn't in the first message. |
| 2 | 1-B | Claude Code + Opus 5.5 | `prompts/1-recover-architecture.md` run B, evidence `protos/demo.proto` + `kubernetes-manifests/` | `prompts/1-recover-architecture.md`, `AGENTS.md` (auto-loaded), `system/online-boutique/protos/demo.proto`, all 13 files in `system/online-boutique/kubernetes-manifests/` (as reported in `outputs/1B-architecture.md` §2) | Yes. Opened only the task file and the evidence. | (you fill) | Output: `exercise/outputs/1B-architecture.md`. 12 containers (incl. `redis-cart`, `loadgenerator`), 17 deps from `*_ADDR` env vars. Languages not evidenced. `shoppingassistantservice` referenced by frontend but no manifest, so listed under Gaps. |
| 3 | 2-a | Claude Code + Opus 5.5 | `prompts/2-write-adr.md` run a, context `none`, requirement R-12 pasted, language English | `docs/adr/template.md`, `prompts/2-write-adr.md`, `exercise/02-write-adr.en.md`, `AGENTS.md` (auto-loaded) | **Partly no.** It didn't open `docs/scenario.md`, but it read the worksheet before the run (same session). The worksheet's comparison table mentions "exactly once" loyalty points and MailFast's rate limit, and both topics show up in 2a as open questions and risks. It wasn't a fresh session either. | (you fill) | Output: `exercise/outputs/2a-adr.md`. Picked broker pub/sub (no product named), with outbox as follow-up. No numbers invented. Unknowns went to Open questions. Consider rerunning in a clean session. |
| 4 | 2-b | Claude Code + Opus 5.5 (per header of `2b-adr.md`) | `prompts/2-…` run b | (you fill, from that session's summary) | (you fill) | (you fill) | Output: `exercise/outputs/2b-adr.md` (run in a separate session). Its title picks RabbitMQ. |
| 5 | 3 | | `prompts/3-…` | | | | |
| 6 | (meta) | Claude Code + Opus 5.5 | "add every logs using this template `exercise/00-prompt-log.en.md`" | `exercise/00-prompt-log.en.md`, top of `outputs/1A-architecture.md` and `outputs/2b-adr.md` (agent line + files-read section only) | n/a (not an evidence task) | (you fill) | Produced this file. |
| 7 | | | | | | | |

