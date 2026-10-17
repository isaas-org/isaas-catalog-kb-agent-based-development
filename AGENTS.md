# Knowledge base conventions

This repo is the knowledge base (KB) for iSaaS, an "Amazon for SaaS" marketplace. It records, per repo and per feature, what was built, the decisions behind it, and the next steps.

## Layout

```
<repo-name>/<feature-folder>/<doc-name>-YYYY-MM-DD-HH-MM.md
isaas-platform/language-and-framework/<doc-name>-YYYY-MM-DD-HH-MM.md
isaas-platform/architecture/<doc-name>-YYYY-MM-DD-HH-MM.md
```

- Folder names are chosen by the repo owner. Always ask before creating a new folder; never invent one.
- File names are free to choose, as long as they are short, kebab-case and end in `-YYYY-MM-DD-HH-MM.md`.
- The timestamp is when the documented work or decision happened, in UTC.

## Writing a doc

- Each doc covers what was done, the decisions behind it (options considered and why one was picked), and the next steps.
- Keep a doc under 300-350 lines or 7 `##` sections, whichever comes first. Split into more docs rather than going over.
- Where code is involved, put short snippets next to the explanation they support. Examples are a DTO and its mapping, a changed service method, a validation rule, or a SQL constraint. Show only the lines that carry the decision, not whole files.
- Record facts as they are in the code, and point out where the code differs from an earlier plan.
- Plain prose and tables. No emojis.

## Working on this repo

- Always work on a feature branch. Never commit or push to `main`.
- Use the branch name the repo owner gives; the pattern is `feature/FD-<n>_<Description>`.
- Merging to `main` is done by the repo owner.
