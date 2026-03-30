# Send-back Helper

You are executing a human-triggered send-back for the OldToby ideation pipeline. The operator wants to send a promoted idea back into the pipeline for another pass with their feedback. Do not ask for input. Do not pause. Execute all steps in order.

---

## Step 1 — Locate the Idea

Look for the folder at `review/$ARGUMENTS/`.

- **If found:** proceed to Step 2.
- **If not found:** check `pipeline/$ARGUMENTS/`.
  - If the idea exists in `pipeline/`: report that the idea is already in the pipeline and **abort**. Do not continue.
  - If the idea is not found in either location: report that the idea `$ARGUMENTS` was not found anywhere and **abort**. Do not continue.

---

## Step 2 — Verify Feedback File

Check if `review/$ARGUMENTS/feedback.md` exists.

- **If it exists:** read the file and confirm it has content. This is operator feedback that the PM agent will use during re-evaluation.
- **If it does not exist:** print a WARNING that the PM agent will re-evaluate this idea without specific operator guidance. Proceed anyway — feedback is optional.

Note whether feedback is `present` or `missing` for the summary in Step 6.

---

## Step 3 — Move Idea to Pipeline

Move the entire folder from `review/$ARGUMENTS/` to `pipeline/$ARGUMENTS/`:

1. Create `pipeline/$ARGUMENTS/` if it does not already exist.
2. Copy all files from `review/$ARGUMENTS/` into `pipeline/$ARGUMENTS/`.
3. Verify the copy succeeded — confirm all files are present in the destination.
4. Delete the `review/$ARGUMENTS/` folder and its contents.

If any step fails, report the error and **abort**. Do not leave the idea in a partially-moved state.

---

## Step 4 — Update Frontmatter

Edit `pipeline/$ARGUMENTS/one-pager.md` frontmatter:

- Set `status: brainstormed`
- Increment `iteration` by 1 (read the current value first, then add 1)
- Do **NOT** change `slug`, `generated`, `keywords`, or any other frontmatter fields

---

## Step 5 — Update Idea Registry

Append a single row to `logs/idea-registry.md`:

```
| {slug} | {title} | {problem} | {keywords} | re-entered | {YYYY-MM-DD} | Human send-back |
```

- `slug`: the idea slug (`$ARGUMENTS`)
- `title`: from the one-pager frontmatter
- `problem`: the one-line problem statement from the one-pager
- `keywords`: from the one-pager frontmatter
- `YYYY-MM-DD`: today's date

**Append only** — do not modify or rewrite any existing rows in the registry.

---

## Step 6 — Output Summary

Print the following summary:

```
Send-back complete.
Idea: {slug}
Moved: review/{slug}/ → pipeline/{slug}/
Status: brainstormed
Iteration: {new iteration number}
Feedback: {present | missing (warning issued)}
Registry: updated
```
