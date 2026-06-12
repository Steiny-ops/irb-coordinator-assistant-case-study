# IRB Application Review Assistant

A case study: building an AI-assisted document review tool for a university research compliance office, run in production on real submissions, with the architecture decisions that made it trustworthy enough for compliance work.

This repository documents the design and the engineering lessons. It contains no institutional data, no real applications, no prompts, and no operational configuration. Names, identifiers, and institution-specific details are omitted or genericized.

## The problem

A university IRB (Institutional Review Board) office reviews human-subjects research applications: long PDFs with required certificates, consent scripts, recruitment materials, and dozens of compliance requirements that must each be checked on every submission. Review was thorough but slow and entirely manual, and revision rounds required re-checking a resubmitted application against every previously identified issue by hand.

Constraints: one builder (a coordinator, not a developer), no budget, no server, and a hard requirement that the tool assist the coordinator's judgment rather than replace it.

## The architecture

A single self-contained HTML file. No server, no database, no installation: the tool runs entirely in the browser, which meant it could exist, be used, and prove its value before any infrastructure conversation happened. API calls route through a secured proxy (a Cloudflare Worker) so the API key lives in a secret store and is never present in the HTML or exposed to the browser.

Two tools in one tabbed interface:

**Application Review** uses a two-call LLM architecture:

- **Call 1, extraction:** a fast, inexpensive model reads the entire PDF and returns a structured JSON object of 70+ fields. It performs no judgment; it only reads.
- **Call 2, review:** a more capable model receives the extracted JSON only, never the PDF, and runs a fixed battery of mandatory pre-checks, a field-by-field review, and cross-checks between fields.

Separating extraction from judgment was the central design decision. It cut cost (the expensive model never reads hundred-page PDFs), made failures diagnosable (an error is traceable to either bad extraction or bad reasoning, never an opaque blend), and made the review auditable (the JSON is a human-readable intermediate that the coordinator can inspect and correct).

**Revision Checker** runs the same two-call pattern against a resubmission: the coordinator uploads the revised PDF alongside the JSON exported from the original review, and each previously identified requirement comes back as ADDRESSED, PARTIAL, or OUTSTANDING. The export captures the coordinator's *edited* findings, not the model's raw output, so the revision round inherits human judgment, not just machine output.

A side tool generates a fully populated project-management card from a submitted application using extraction only, at a cost of roughly one cent per card.

## Trust engineering

The features that made the tool fit for compliance work are the ones that constrain the AI:

- **Deterministic guardrails the model cannot override.** For requirements that must never be missed (for example, that the researcher's own contact information appears in consent scripts), a plain JavaScript check runs after the AI and overrides it. The model proposes; code disposes.
- **Human override as a first-class interface.** Every finding carries accept / partial / reject controls and a remove-restore flow. The tool's output is the coordinator's reviewed judgment, with the AI as a first-pass reader.
- **An explicit governance document.** The office adopted written rules for the tool: it assists and does not decide, categories of sensitive material are excluded from processing entirely, and its use was disclosed to the institution's compliance and IT functions.

## Production history

The tool ran on real submissions, with per-application accuracy review driving prompt iteration: each real case that exposed a miss became a named fix. The security architecture evolved in deliberate phases (direct API calls, then the proxy), hosting moved from a personal-tier platform to institutional infrastructure in coordination with campus IT, and the decision history is kept as architecture decision records and a changelog.

That last part is its own lesson: a tool becomes institutional not when it works but when it stops depending on its builder's personal accounts, and that migration is real engineering work worth planning from the start.

## Results

A production tool that turned a multi-hour manual document review into a first-pass that takes minutes, with the coordinator's judgment preserved at every step; a revision-checking workflow that re-verifies every prior finding automatically; and a written governance model that let a compliance office adopt AI assistance on its own terms.

See [docs/lessons-learned.md](docs/lessons-learned.md) for the engineering lessons.

## Author

Built and documented by Spencer Steinberg. This case study is personal work product describing generalizable architecture and lessons; it intentionally contains no institutional information.
