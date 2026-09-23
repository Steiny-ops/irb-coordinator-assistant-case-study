# IRB Coordinator Assistant

A case study of an AI-assisted review tool built for a university research compliance office. It was used in production on real applications. This write-up covers how it was built and what made it reliable enough for compliance work.

This repository has no institutional data, real applications, prompts, or configuration. Names, identifiers, vendors, and institution-specific details are left out or made generic.

## The problem

A university IRB (Institutional Review Board) office reviews research applications that involve human subjects. Each application is a long PDF with certificates, consent scripts, recruitment materials, and dozens of requirements that have to be checked every time. The review was careful but slow, and all of it was done by hand. When a researcher sent back a revised application, every earlier issue had to be checked again by hand too.

The constraints: one builder (a coordinator, not a developer), no budget, no server, and the tool had to support the coordinator's judgment, not replace it.

## How it's built

The whole tool is one HTML file. There is no application server, no database, and nothing to install. It runs in the browser, so it could be built, used, and shown to work before anyone had to approve infrastructure for it. Calls to the AI go through a secured proxy on an edge worker. The API key is kept in a secret store there and never appears in the HTML or the browser.

There are two tools in one tabbed interface.

**Application Review** makes two AI calls:

- **Call 1, extraction:** a fast, low-cost model reads the whole PDF and returns 70+ fields as structured data. It only reads. It doesn't judge anything.
- **Call 2, review:** a stronger model gets the extracted data, not the PDF. It runs a fixed set of required checks, reviews each field, and cross-checks fields against each other.

Splitting reading from judging was the most important design choice. It kept costs down, because the expensive model never has to read a hundred-page PDF. It made problems easy to trace, because any mistake comes from either the extraction or the review, not a mix of both. And it made the review easy to audit, because the extracted data is readable and the coordinator can check and correct it.

**Revision Checker** uses the same two calls on a resubmitted application. The coordinator uploads the revised PDF along with the results saved from the first review, and each earlier issue comes back marked ADDRESSED, PARTIAL, or OUTSTANDING. The saved results are the coordinator's edited findings, not the AI's original output, so the second round starts from what the coordinator actually decided.

A smaller tool creates a filled-in project board card from an application using only the extraction step. Each card costs about one cent.

## Keeping a person in charge

The features that make the tool safe to use for compliance work are the ones that limit what the AI can do:

- **Checks the AI can't override.** For requirements that can never be missed, like the researcher's contact information appearing in the consent script, a simple JavaScript check runs after the AI and has the final say.
- **The coordinator can override everything.** Every finding has accept, partial, and reject buttons, and findings can be removed and restored. What comes out of the tool is the coordinator's reviewed judgment. The AI only does the first read.
- **Drafts only.** The tool drafts the letter back to the researcher, and the coordinator reviews every draft before it's sent. The tool never makes a decision and never sends anything by itself.
- **Written rules.** The office adopted written rules for using the tool. It assists and doesn't decide, certain kinds of sensitive material are never sent to it, and its use was disclosed to the institution's compliance and IT offices.

## How accurate it is

Accuracy was measured on real applications, not estimated. For each application, every result the tool produced was checked by hand against the source documents and marked right or wrong.

On a typical application the tool runs about 150 checks and gets one or two wrong, which is about 99% accuracy.

The same method was used before upgrading to a newer AI model. Both models were run on the same real applications and every flag was checked. With the newer model, the share of flagged problems that turned out to be wrong dropped from about 36% to about 13%, so the upgrade went ahead.

These two numbers measure different things. The 99% covers every check, and most checks pass. The flag rate only covers the problems the tool raises, which are what the coordinator actually has to act on.

## Mistakes it makes

Every mistake found in production got its own fix, and the testing was run again afterward. Three kinds of mistakes came up often enough to name, and anyone building something similar should expect them:

- **Made-up problems:** the tool flags an issue that isn't in the document.
- **Mixing up documents:** an application comes as several attachments, and the tool credits something to the wrong file or misses that a requirement is met in a different one.
- **Made-up rules:** the tool cites a requirement that sounds right but isn't one the office actually has.

None of these can be fully eliminated. That's why the coordinator's review is required on everything.

## Moving it into production

The tool was used on real applications, and reviewing its accuracy on each one drove the changes to the prompts. Every real case that exposed a miss got a named fix. The security setup was built in stages, first with direct API calls and then with the proxy. With campus IT, the tool moved from a personal hosting account to institutional hosting and campus-managed source control, and it went through an institutional security review. Every major design decision is recorded, along with a changelog.

A tool isn't really the institution's until it stops depending on the builder's personal accounts. That move takes real work and is worth planning from the start.

## Why switching AI providers isn't simple

It's easy to think of the AI provider as a setting you can change by pointing the proxy somewhere else. For this tool it isn't, for two reasons.

The first is technical. The tool depends on features of the model it was built on, like reading a PDF directly and returning answers in a fixed format. Every check is built around those features, so a new provider means rebuilding every check.

The second is more important. Every check and prompt was tuned for one model, and the accuracy numbers above only apply to that model. With a different model, the tool would be running on compliance work with an error rate nobody has measured. Reconnecting it is quick. Retesting it properly is the real job.

## Results

In production, the tool cut coordinator time per application by about two-thirds and brought researcher turnaround from about a week down to the same day. A review that took hours by hand became a first pass that takes minutes. The coordinator's corrections to outgoing letters dropped noticeably once the tool was in use. Resubmissions are checked against every earlier finding automatically, and the written rules gave the office a way to use AI that it was comfortable with.

The engineering lessons are in [lessons-learned.md](lessons-learned.md).

## Author

Built and documented by Spencer Steinberg. This case study is my own work and describes general architecture and lessons. It contains no institutional information.
