# How to Edit & Maintain Your Course Website

## Quick Reference

**Repo:** https://github.com/vkoul/ab-test-practice  
**Live site:** https://vkoul.github.io/ab-test-practice/  
**Local project:** `C:\Users\vkoul\projects\ab-testing-course`  
**Auto-deploy:** Every push to `main` triggers a rebuild (~2 min)

---

## The Basic Workflow

```bash
cd C:\Users\vkoul\projects\ab-testing-course

# 1. Make your edits to any file
# 2. Stage, commit, push
git add -A
git commit -m "describe your change"
git push
```

That's it. GitHub Actions will rebuild and redeploy the site automatically.

---

## What File Controls What

| You want to change... | Edit this file |
|---|---|
| Landing/home page | `intro.md` |
| Study notes (Week 1-4) | `study_notes/week0X_study_*.ipynb` |
| Assignments (Week 1-4) | `assignments/week0X_assignment_*.ipynb` |
| Solutions (Week 1-4) | `solutions/w0X_*.ipynb` |
| Cheat sheet | `reference/ab_testing_cheat_sheet.md` |
| Course notes | `reference/course_notes.md` |
| Concept map | `reference/concept_map.md` |
| A/B Testing lesson | `reference/ab_testing_lesson.md` |
| 30 Questions | `reference/30_critical_questions.ipynb` |
| **Navigation sidebar** | `myst.yml` → `toc:` section |
| **Site title & metadata** | `myst.yml` → `project:` and `site:` sections |
| **Build/deploy process** | `.github/workflows/deploy.yml` |
| **Python dependencies** | `requirements.txt` |

---

## Editing Markdown Files (.md)

Open in any text editor. Uses standard Markdown syntax:

```markdown
# Heading 1
## Heading 2
### Heading 3

**bold** and *italic*

- bullet points
- like this

| Column 1 | Column 2 |
|----------|----------|
| data     | data     |

```python
code blocks with syntax highlighting
```​

[link text](https://url.com)

![image alt](path/to/image.png)
```

### MyST Markdown extras (supported by Jupyter Book)

```markdown
:::{note}
This renders as a highlighted note box.
:::

:::{warning}
This renders as a warning box.
:::

:::{tip}
This renders as a tip box.
:::

:::{admonition} Custom Title
:class: hint
Custom admonition with any title.
:::
```

---

## Editing Notebooks (.ipynb)

**Option 1: Jupyter Lab** (recommended for heavy edits)
```bash
cd C:\Users\vkoul\projects\ab-testing-course
jupyter lab
```

**Option 2: VS Code** — open the .ipynb file directly, it renders as a notebook.

**Option 3: Google Colab** — open from GitHub, edit, then download and replace the local file.

**Option 4: Ask me** — tell me what to change and I'll edit it for you.

### Tips for notebook edits:
- Markdown cells render as page content
- Code cells render with syntax highlighting (but won't execute on the website unless you run them first and save with outputs)
- To show code OUTPUT on the website: run the cell locally before committing so the output is saved in the .ipynb file
- Empty output cells just show the code block without results

---

## Adding a New Page

1. Create the file in the appropriate folder:
   - `.md` for text/reference content
   - `.ipynb` for notebook content with code

2. Add it to `myst.yml` under the correct section:
```yaml
    - title: "Reference"
      children:
        - file: reference/my_new_page        # no extension needed
          title: "My New Page Title"
```

3. Commit and push.

---

## Removing a Page

1. Delete the file from disk
2. Remove its entry from the `toc:` section in `myst.yml`
3. Commit and push

---

## Reordering Pages

Just reorder the entries in `myst.yml` under `toc:`. The sidebar follows the order you specify.

---

## Adding a New Section (e.g., "Extras")

Add a new group in `myst.yml`:
```yaml
    - title: "Extras"
      children:
        - file: extras/my_page
          title: "My Extra Page"
```

Create the folder and file, commit, push.

---

## Adding Images

1. Put the image in a folder (e.g., `images/my_diagram.png`)
2. Reference in markdown: `![description](images/my_diagram.png)`
3. Or in a notebook markdown cell: same syntax
4. Commit the image file along with your changes

---

## Controlling What Shows on the Site

### Hide a page temporarily (without deleting)
Just comment it out in `myst.yml`:
```yaml
    # - file: reference/draft_page
    #   title: "Work in Progress"
```

### Make code cells collapsible
In a notebook, add cell metadata:
```json
{
  "tags": ["hide-input"]
}
```
Options: `hide-input` (hides code, shows output), `hide-output`, `hide-cell` (hides everything)

---

## Troubleshooting

### "Site not loading correctly" / BASE_URL error
The `BASE_URL` is set in `.github/workflows/deploy.yml`:
```yaml
run: BASE_URL=/ab-test-practice jupyter-book build --html
```
If you rename the repo, update this value to match.

### Build fails in GitHub Actions
1. Go to https://github.com/vkoul/ab-test-practice/actions
2. Click the failed run → click the failed job → read the error
3. Common issues:
   - File referenced in `myst.yml` doesn't exist → fix the path or remove the entry
   - Invalid notebook JSON → open and re-save in Jupyter
   - Syntax error in `myst.yml` → check YAML indentation

### Changes not appearing after push
- Wait 2-3 minutes for the build to complete
- Check Actions tab for build status
- Hard-refresh browser (Ctrl+Shift+R) to bypass cache

### Want to preview locally before pushing
```bash
cd C:\Users\vkoul\projects\ab-testing-course
jupyter-book build --html
# Open _build/html/index.html in browser
```
(Note: requires npm network access which may be blocked on your corporate network)

---

## Useful Git Commands

```bash
# Check what you've changed
git status
git diff

# Undo changes to a file (before committing)
git checkout -- path/to/file

# See recent commits
git log --oneline -10

# Push a specific file only
git add path/to/file
git commit -m "update specific file"
git push
```

---

## Pro Tips

1. **Commit often, push often** — small changes are easier to debug if the build breaks.

2. **Write meaningful commit messages** — you'll thank yourself later:
   - Good: "Add novelty effect example to week 3 study notes"
   - Bad: "update"

3. **Test notebooks locally** before pushing — if a notebook has errors in its JSON structure, the build will fail.

4. **Keep data files in `data/`** — notebooks reference them via `../data/filename.csv`. If you add new datasets, put them there.

5. **The site rebuilds on EVERY push** — even a typo fix in a README triggers a full rebuild. This is fine (it's fast), just be aware.

6. **To update notebook data paths for Colab** — users opening in Colab won't have `../data/` available. Consider adding a cell at the top of notebooks that downloads data from the raw GitHub URL:
   ```python
   # For Colab users:
   # !wget https://raw.githubusercontent.com/vkoul/ab-test-practice/main/data/dataset_autotitle.csv
   ```

7. **Custom domain** — if you ever want `abtesting.yourdomain.com` instead of `vkoul.github.io/ab-test-practice`, add a `CNAME` file to the repo root with your domain, and configure DNS to point to GitHub Pages.
