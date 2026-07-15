# Claude Skill Library

A personal collection of custom [Claude Skills](https://docs.claude.com) — specialized expert-persona instruction sets that give Claude domain-specific workflows, output formats, and behavioral rules for recurring tasks in DevOps/SRE, learning, travel planning, image work, and job applications.

Each skill lives in its own folder as a `SKILL.md` file (plus optional bundled resources) and is triggered automatically when a request matches its description — no manual invocation needed.

## Skills

| Skill | What it does |
|---|---|
| **[cicd-pipeline-architect](./cicd-pipeline-architect)** | Elite CI/CD automation architect for Jenkins, Groovy, Bash, and Python. Runs a strict four-phase workflow (Discovery → Risk Assessment → Code Generation → Testing) for writing/reviewing Jenkinsfiles, debugging pipeline failures, hardening credentials, and fixing flaky builds. |
| **[iac-architect](./iac-architect)** | Full-lifecycle Infrastructure-as-Code framework for Terraform, Terragrunt, OpenTofu, and Crossplane across AWS/Azure/GCP. Covers authoring, drift reconciliation, state surgery, IAM policy design, and cost guarding for multi-contributor environments. |
| **[k8s-guardian](./k8s-guardian)** | Full-lifecycle Kubernetes debugger and guardian. Diagnoses pod/node failures (OOMKilled, CrashLoopBackOff, Evicted), catches silent failures (selector mismatches, stale DNS, dead endpoints), runs security audits, and manages AI/GPU workloads (NVIDIA device plugin, vLLM, KServe). |
| **[system-architect-bottleneck-debugger](./system-architect-bottleneck-debugger)** | Holistic system analysis and bottleneck identification across distributed systems, cloud infra, databases, and application performance. Includes an INCIDENT MODE for active SEV-1/SEV-2 outages that returns only copy-paste fix commands. |
| **[production-readiness-mentor](./production-readiness-mentor)** | Produces battle-tested ramp-up roadmaps with time estimates, hands-on exercises, and production pitfall analysis for any tool, platform, or system a senior SRE/DevOps engineer needs to operate at scale. |
| **[practical-skill-blueprint](./practical-skill-blueprint)** | Generates self-sufficient, layman-friendly learning blueprints for any topic — zero legacy bloat, textual diagrams, production code snippets, and edge-case coverage thorough enough to replace external tutorials. |
| **[travel-concierge-logistics](./travel-concierge-logistics)** | Real-time travel concierge that builds end-to-end couple's itineraries with live train/bus schedules, hotel availability, and restaurant picks — "smart-economical" philosophy, prices in INR with FX conversion. |
| **[image-gen-editing-expert](./image-gen-editing-expert)** | Expert prompt engineer for Midjourney v6 and high-end photo editing (Photoshop, Lightroom, GIMP). Delivers exact prompt syntax and precise editing steps for character consistency, upscaling, masking, and retouching. |
| **[job-application-optimizer](./job-application-optimizer)** | Tailors resumes to job postings, generates customized cover letters, and prepares role-specific interview questions based on job description analysis. |

## Structure

```
.
├── README.md
├── cicd-pipeline-architect/
│   └── SKILL.md
├── iac-architect/
│   └── SKILL.md
├── k8s-guardian/
│   └── SKILL.md
├── system-architect-bottleneck-debugger/
│   └── SKILL.md
├── production-readiness-mentor/
│   └── SKILL.md
├── practical-skill-blueprint/
│   └── SKILL.md
├── travel-concierge-logistics/
│   └── SKILL.md
├── image-gen-editing-expert/
│   └── SKILL.md
└── job-application-optimizer/
    └── SKILL.md
```

Every `SKILL.md` starts with YAML frontmatter (`name`, `description`) followed by the instruction body. The `description` field is the triggering mechanism — Claude reads it to decide when to consult the skill, so it's written to be specific and a little "pushy" about when it applies.

## Using these skills

**Claude.ai / Claude apps:** Upload or paste a `SKILL.md` file, or use the "Save skill" button when a skill is generated in a conversation with skill-creation tooling.

**Claude Code / Cowork:** Drop the skill folder into your skills directory (commonly `/mnt/skills/user/` or your project's configured skills path) so it's picked up automatically.

**API:** Skills can be supplied as part of the system prompt or loaded via whatever skill-loading mechanism your integration uses — see [Anthropic's docs](https://docs.claude.com) for current specifics.

## Editing a skill

Each `SKILL.md` is plain Markdown with YAML frontmatter — edit directly. Keep in mind:
- `description` has a hard length limit (1024 characters) and is the only thing always in context, so front-load the triggering conditions.
- Keep the body under ~500 lines where possible; split overflow into a `references/` subfolder and point to it from the main file.
- After editing, re-validate before re-deploying (if using the skill-creator tooling: `python -m scripts.quick_validate <skill-folder>`).

## Contributing / extending

This is a personal, evolving library. New skills get added as recurring workflows emerge; existing ones get refined based on real output quality (see `practical-skill-blueprint`'s revision history for an example of iterating a skill after reviewing its actual generated output).