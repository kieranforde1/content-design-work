# Content files

These Markdown files are the source of truth for site content. Edit them here, then hand back to regenerate the HTML.

## Structure

```
content/
  profile.md                  — bio, intro text, profile photo
  projects/                   — one file per summary card
    use-cases.md
    terminology.md
    ai-experiences.md
    ons-bulletins.md
    trim.md
    ons-data-tools.md
  case-studies/               — one file per full case study page
    use-cases.md
    terminology.md
    ai-experiences.md
    ons-bulletins.md
    trim.md
    ons-data-tools.md
```

## How to edit

- Edit any `.md` file in a text editor or directly in GitHub
- YAML frontmatter (the block between `---` lines at the top) holds structured fields like title, tags, year
- Body text is plain Markdown — `**bold**`, `_italic_`, paragraphs separated by blank lines
- Outcome stats use a simple `stat:` list in the frontmatter
- Image placeholders are marked with `<!-- IMAGE: description (slide N) -->` — replace with actual filenames once images are exported

## Regenerating the HTML

Once you've made edits, share the updated files and the HTML will be regenerated to match.
