# Claude Skill Library

A personal collection of custom [Claude Skills](https://docs.claude.com) — specialized expert-persona instruction sets that give Claude domain-specific workflows, output formats, and behavioral rules for recurring tasks in DevOps/SRE, learning, travel planning, image work, and job applications.

Each skill lives in its own folder as a `SKILL.md` file (plus optional bundled resources) and is triggered automatically when a request matches its description — no manual invocation needed.

## Skills

| Skill | What it does |
|---|---|
| **[cicd-pipeline-architect](./cicd-pipeline-architect.skill)** | Elite CI/CD automation architect for Jenkins, Groovy, Bash, and Python. Runs a strict four-phase workflow (Discovery → Risk Assessment → Code Generation → Testing) for writing/reviewing Jenkinsfiles, debugging pipeline failures, hardening credentials, and fixing flaky builds. |
| **[iac-architect](./iac-architect.skill)** | Full-lifecycle Infrastructure-as-Code framework for Terraform, Terragrunt, OpenTofu, and Crossplane across AWS/Azure/GCP. Covers authoring, drift reconciliation, state surgery, IAM policy design, and cost guarding for multi-contributor environments. |
| **[k8s-guardian](./k8s-guardian.skill)** | Full-lifecycle Kubernetes debugger and guardian. Diagnoses pod/node failures (OOMKilled, CrashLoopBackOff, Evicted), catches silent failures (selector mismatches, stale DNS, dead endpoints), runs security audits, and manages AI/GPU workloads (NVIDIA device plugin, vLLM, KServe). |
| **[system-architect-bottleneck-debugger](./system-architect-bottleneck-debugger.skill)** | Holistic system analysis and bottleneck identification across distributed systems, cloud infra, databases, and application performance. Includes an INCIDENT MODE for active SEV-1/SEV-2 outages that returns only copy-paste fix commands. |
| **[production-readiness-mentor](./production-readiness-mentor.skill)** | Produces battle-tested ramp-up roadmaps with time estimates, hands-on exercises, and production pitfall analysis for any tool, platform, or system a senior SRE/DevOps engineer needs to operate at scale. |
| **[practical-skill-blueprint](./practical-skill-blueprint.skill)** | Generates self-sufficient, layman-friendly learning blueprints for any topic — zero legacy bloat, textual diagrams, production code snippets, and edge-case coverage thorough enough to replace external tutorials. |
| **[travel-concierge-logistics](./travel-concierge-logistics.skill)** | Real-time travel concierge that builds end-to-end couple's itineraries with live train/bus schedules, hotel availability, and restaurant picks — "smart-economical" philosophy, prices in INR with FX conversion. |
| **[image-gen-editing-expert](./image-gen-editing-expert.skill)** | Expert prompt engineer for Midjourney v6 and high-end photo editing (Photoshop, Lightroom, GIMP). Delivers exact prompt syntax and precise editing steps for character consistency, upscaling, masking, and retouching. |

> **Legacy:** `cicd-failure-analyzer.md` is an earlier version of the CI/CD skill, superseded by `cicd-pipeline-architect.skill`. Kept for reference; safe to remove once you've confirmed you no longer need it.

## Structure

Flat repo — each skill is a single packaged `.skill` file at the root (no subfolders):

```
.
├── README.md
├── LICENSE
├── cicd-pipeline-architect.skill
├── iac-architect.skill
├── k8s-guardian.skill
├── system-architect-bottleneck-debugger.skill
├── production-readiness-mentor.skill
├── practical-skill-blueprint.skill
├── travel-concierge-logistics.skill
├── image-gen-editing-expert.skill
└── cicd-failure-analyzer.md   # legacy, superseded — see note above
```

A `.skill` file is a packaged bundle containing `SKILL.md` (YAML frontmatter + instructions) and any bundled resources. Unzip it if you need to inspect or edit the raw `SKILL.md`.

## Using these skills

**Claude.ai / Claude apps:** Download the `.skill` file and use the "Save skill" button/upload prompt to install it into your profile.

**Claude Code / Cowork:** Unzip the `.skill` file (it's a zip archive containing `SKILL.md` and any bundled resources) into your skills directory (commonly `/mnt/skills/user/<skill-name>/`) so it's picked up automatically.

**API:** Extract `SKILL.md` from the archive and supply it as part of the system prompt or via whatever skill-loading mechanism your integration uses — see [Anthropic's docs](https://docs.claude.com) for current specifics.

## Editing a skill

Each `.skill` file is a zip archive — unzip it to get the editable `SKILL.md` (plain Markdown with YAML frontmatter):

```bash
unzip cicd-pipeline-architect.skill -d cicd-pipeline-architect/
# edit cicd-pipeline-architect/SKILL.md
```

Keep in mind:
- `description` has a hard length limit (1024 characters) and is the only thing always in context, so front-load the triggering conditions.
- Keep the body under ~500 lines where possible; split overflow into a `references/` subfolder and point to it from the main file.
- After editing, re-validate and re-package before committing the updated `.skill` file back (if using the skill-creator tooling: `python -m scripts.quick_validate <folder>` then `python -m scripts.package_skill <folder> <output-dir>`).

## Contributing / extending

This is a personal, evolving library. New skills get added as recurring workflows emerge; existing ones get refined based on real output quality (see `practical-skill-blueprint`'s revision history for an example of iterating a skill after reviewing its actual generated output).