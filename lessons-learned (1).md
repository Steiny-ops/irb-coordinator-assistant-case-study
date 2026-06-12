# Lessons Learned

From building, operating, and institutionalizing an AI-assisted document review tool for compliance work. Each lesson cost real time or was earned against real submissions.

## LLM architecture

**1. Separate extraction from judgment.**
One model call that reads a long PDF and renders findings is opaque, expensive, and undebuggable. Split it: a fast, cheap model extracts the document into structured JSON (no judgment), then a capable model reasons over the JSON alone (no document). Failures become diagnosable, the expensive model never pays to read raw pages, and the intermediate JSON is an audit artifact a human can inspect and correct.

**2. Token budgets are design constraints, not afterthoughts.**
Fixed output ceilings per call (a small budget for extraction, a large one for the full review) force the prompt design to prioritize. Decide what the model must always have room to say, and budget backward from that.

**3. Structured output is a contract.**
When the reviewing model consumes extraction JSON and the interface maps findings to controls, the field schema is an API between calls. Treat schema changes with the same care as code changes; a renamed field is a silent break two steps downstream.

**4. Iterate prompts against named real cases, not intuitions.**
Accuracy improved by reviewing the tool's output on individual real submissions, scoring it, and turning each miss into a specific prompt fix tied to that case. A regression set of real (sanitized) documents is worth more than any amount of prompt theorizing.

## Trust and guardrails

**5. For must-never-miss checks, run code after the AI.**
Some requirements are too important to entrust to a probabilistic reader. Implement them as deterministic post-processing that can override the model's finding but never be overridden by it. The model proposes; code disposes.

**6. Guardrail code needs adversarial testing too.**
A deterministic check produced a false positive by accepting a related party's contact information where the researcher's own was required, and a false negative risk by letting one field's content satisfy another field's check. Guardrails earn trust the same way the AI does: by being tested against the cases meant to fool them, and by isolating checks per field so content cannot cross-contaminate.

**7. Make human override a first-class interface, and export the human's state.**
Accept / partial / reject controls on every finding keep the coordinator the author of the review. Critically, anything exported downstream (to the revision-checking round) must capture the coordinator's edited findings, not the model's raw output; otherwise the next round inherits the machine's mistakes after a human already fixed them.

**8. Govern the AI in writing before someone asks you to.**
A short adopted document, the tool assists and does not decide, these material categories are excluded from processing, here is who was told, converts an experiment into an institutional practice. It also forces the design conversation (what should this tool refuse to touch?) at the right time, which is early.

## Security

**9. The API key never touches the browser.**
A client-side tool cannot hold a secret. Route calls through a proxy (a serverless worker) holding the key in a secret store. This was an evolution, not a day-one design, and the interim states were the riskiest period of the project; if a tool will live longer than a demo, build the proxy first.

**10. Single-file architecture is a deployment superpower with a ceiling.**
One HTML file meant the tool could run anywhere, instantly, with no infrastructure approval, which is how it got to exist at all. The ceiling appeared when two tools merged into one file: global CSS variables collided, function names collided, and initialization order bugs emerged. The fixes (prefix-namespacing all of one tool's functions, scoping its CSS under one container) are manual disciplines that a build system gives you for free. Know which side of that trade you are on, and when you cross it.

## Operations and institutionalization

**11. Personal infrastructure is a liability the moment a tool matters.**
Hosting on a personal-tier platform and building under an individual account is how solo tools start, and it becomes the single biggest institutional risk they carry. Migrating to institution-managed hosting, in coordination with campus IT, was real engineering work with its own decision records. Plan the handoff from the start: the goal is a tool that survives its builder.

**12. Keep architecture decision records and a changelog.**
Compliance offices reason in documents. A numbered decision record for each significant choice (proxy architecture, hosting migration) and a changelog of what changed when turned out to be the artifacts that made IT and leadership conversations easy, because the history was already written down, with reasons.

**13. Cost-engineer the boring automations.**
Extraction-only calls with a cheap model made a fully populated project-management card cost about one cent. Not every AI feature needs the expensive model; matching model capability to task tier is where per-unit economics become negligible.

**14. Production trust is earned by use, not promised by design.**
The tool's standing came from processing real submissions and being right, visibly, with its misses caught by the human-in-the-loop design and fixed in the open. For AI tools in conservative institutions especially: ship something honest and assistive, run it on real work, and let the track record make the argument.
