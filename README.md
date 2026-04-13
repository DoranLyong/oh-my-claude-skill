# oh-my-claude-skills
Customized Claude Code Skills



---

Extracting the chapter contents that you want to organize from your book:

```bash
/extract-chapter <book_name> <chapter_number> <output_filename>.md

#-- Example
/extract-chapter @<book_name> chapter4 Geometry_04.md
```

---

Generating lecture notes from a video file with frame extraction and speech transcription:

```bash
Generate a study note using /video-to-note. Whisper transcription may take 30+ minutes — wait for it to finish completely. Do not abort or retry.
```

---

Generating study notes from a paper.

```bash
/paper-to-note read @new_paper.pdf and create a study_note.md
```

---

Generating a project overview from both paper and project code. 

```
/project-to-note create a project overview based on @new_paper.pdf and the current codebase
```

Generating an interactive code review. 

```
/gen-codeflow @PATH/TO/CODE
```
