# AI-200 Study Notes

Personal study notes for the Microsoft Azure **AI-200** certification exam.

> Align the folders in `notes/` with the skills outline in the official
> [AI-200 study guide](https://learn.microsoft.com/credentials/) and update them as the outline changes.

## Structure

| Folder | Purpose |
| --- | --- |
| [`notes/`](notes/) | Topic notes, one Markdown file per concept |
| [`labs/`](labs/) | Hands-on lab write-ups, commands, and code snippets |
| [`resources/`](resources/) | Links to docs, Learn modules, videos, and practice exams |
| [`templates/`](templates/) | Templates for new notes and labs |
| [`assets/images/`](assets/images/) | Screenshots and diagrams referenced by notes |

## Progress

See [PROGRESS.md](PROGRESS.md).

## Workflow

```bash
cp templates/note-template.md notes/<topic-name>.md   # start a new note
git add . && git commit -m "Add notes on <topic>"     # save
git push                                              # back up to GitHub
```
