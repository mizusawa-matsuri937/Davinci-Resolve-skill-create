# DaVinci Primary Batch Skill Design

Date: 2026-07-05

## Goal

Create a Codex skill for conservative, high-efficiency batch primary color correction in DaVinci Resolve.

The skill uses a hybrid workflow:

- Apply a prepared DRX template for the repeated node tree and LUT setup.
- Use DaVinci Resolve scripting for repeatable batch actions where the API is reliable.
- Use Codex screen inspection for RGB Parade, exposure, overexposure review, and consistency judgment.

The first version focuses on fast, consistent first-pass correction across short video clips with one main subject per clip.

## Default Behavior

- Modify the current color version directly.
- Do not create a new local color version unless the user explicitly asks for rollback safety.
- Process the currently selected timeline clips by default.
- Accept a user-described clip range when provided.
- Use each target clip's first frame as its reference observation frame.
- Use RGB Parade as the default scope for white balance and exposure consistency.
- Use ordinary waveform only as a secondary exposure reference.
- Ask all setup questions once before processing.
- Do not interrupt the user during processing.
- Use conservative correction strength.
- Output one final batch summary only.
- Keep abnormal-condition notes in the final summary only.
- Do not write notes into node labels, clip markers, or DaVinci metadata.

## Out Of Scope

- Complex secondary color correction.
- Automatic power windows or subject masks.
- Automatic outside-node treatment.
- Automatic global style grading.
- Automatic noise reduction parameter changes.
- Strong creative looks or aggressive stylization.

## User Setup Questions

At the start of a batch run, ask all required questions together:

1. Which clips should be processed?
   Default: currently selected timeline clips.
2. What saturation direction should be used?
   Examples: natural, slightly higher, clearly higher, lower.
3. What HDR strength should be used?
   Examples: weak, medium, strong. Default behavior remains conservative.
4. What overall temperature preference should be used?
   Examples: neutral, slightly warm, slightly cool.
5. Is lowering the highlight ceiling allowed when visible overexposure appears?
   Default: allowed.
6. Should the final summary use the default concise format?
   Default: yes.

The skill must not ask for a reference frame. Each clip uses its first frame.

## Node Template

Use `assets/templates/Codex_Primary_Batch_Template_v1.drx` as the first-version node tree template.

Expected node structure:

1. Noise reduction node
   - Create or preserve only.
   - Do not adjust noise reduction settings.
   - If shadows are dominant or shadow noise risk is obvious, record this in the final summary.

2. LUT node
   - Use `SLog3SGamut3.CineToSLog2-709` or the matching configured LUT in the DRX template.
   - If the LUT is missing, stop automatic LUT work and report the issue.

3. Primary node
   - Perform core primary correction.
   - Use RGB Parade to align white balance across key brightness regions.
   - Adjust exposure toward consistent highlight, shadow, and midtone distribution.
   - After waveform/scope adjustment, inspect the image itself.
   - If visible overexposure, dead white areas, or lost subject detail appears, lower the highlight ceiling.

4. Gain node
   - Apply the user's saturation direction.
   - Apply the user's HDR strength conservatively.
   - Slightly reduce highlights and lift shadows when appropriate.
   - Use contrast and pivot to increase separation without pushing the image into an artificial look.

5. Window node
   - Create or preserve only.
   - Do not create or track masks automatically.

6. Outside node
   - Create or preserve only.
   - Do not process automatically.

7. Global adjustment node
   - Create or preserve only.
   - Leave for later manual global style work.

## Batch Workflow

1. Open or switch to the DaVinci Resolve Color page.
2. Collect the setup answers once.
3. Determine the target clips:
   - Prefer currently selected clips.
   - If unavailable through scripting, use the user's described range or UI-driven selection.
4. Apply the DRX template to each target clip.
5. For each clip:
   - Move to the clip's first frame.
   - Inspect the image and RGB Parade.
   - Correct white balance toward batch consistency.
   - Correct exposure toward batch consistency.
   - Apply conservative saturation and HDR adjustments.
   - Recheck the image for visible overexposure.
   - Lower the highlight ceiling if needed and allowed.
   - Record only internal notes for final summary.
6. Continue processing even when one clip cannot be matched cleanly.
7. Output one final batch summary.

## Consistency Criteria

Primary consistency is judged by:

- White balance:
  - RGB channels should be reasonably aligned in key brightness regions.
  - Color temperature controls red/blue bias.
  - Tint controls green/magenta bias.
  - Offset, highlights, shadows, and midtones may be used as needed.

- Exposure:
  - Highlights should be controlled and similar across clips.
  - Shadows should be low enough for contrast but not crushed unnecessarily.
  - Midtones should preserve subject information.
  - Scope targets must be checked against visible image quality.

- Overexposure:
  - Do not rely only on waveform or RGB Parade.
  - If the image shows dead white regions, lost subject texture, or visibly harsh clipping, lower the highlight ceiling.

- Shadows and noise:
  - If shadow-heavy clips show likely noise risk, do not automatically strengthen noise reduction.
  - Record a final-summary note recommending noise reduction review.

- Conservative adjustment:
  - Prefer usable, consistent correction over aggressive look creation.
  - Mark difficult clips for manual review instead of forcing them into an unstable match.

## Final Batch Summary

Output one concise summary after the whole batch finishes.

Include:

- Processing range.
- First-frame reference rule.
- Batch parameters:
  - saturation direction,
  - HDR strength,
  - temperature preference,
  - highlight ceiling policy.
- Common operations performed.
- Clips with visible overexposure risk.
- Clips with shadow noise risk.
- Clips whose white balance or exposure could not be matched cleanly.
- Clips requiring manual review.
- Nodes intentionally left untouched:
  - noise reduction parameters,
  - window node,
  - outside node,
  - global adjustment node.

Do not output a separate summary after each clip.

## Implementation Structure

Target skill structure:

```text
davinci-primary-batch/
├── SKILL.md
├── agents/
│   └── openai.yaml
├── assets/
│   └── templates/
│       └── Codex_Primary_Batch_Template_v1.drx
├── references/
│   ├── workflow.md
│   ├── grading-criteria.md
│   └── resolve-scripting.md
└── scripts/
    ├── apply_template_to_selected.py
    └── collect_batch_context.py
```

Responsibilities:

- `SKILL.md`
  - Triggering description and main workflow.
  - Setup questions.
  - Batch processing order.
  - Summary policy.

- `assets/templates/Codex_Primary_Batch_Template_v1.drx`
  - Fixed DRX template for the first version.
  - Contains the expected node tree.

- `references/workflow.md`
  - Detailed operational workflow.
  - Script-vs-screen boundary.
  - Batch processing sequence.

- `references/grading-criteria.md`
  - Scope-reading and image-inspection rules derived from the XMind notes and user requirements.
  - White balance, exposure, overexposure, shadow noise, and conservative adjustment guidance.

- `references/resolve-scripting.md`
  - DaVinci Resolve scripting capabilities and limits.
  - Document DRX application, timeline/clip context collection, Color page switching, and frame export.
  - Document that internal color node parameters should not be assumed scriptable unless verified.

- `scripts/apply_template_to_selected.py`
  - Attempt to apply the DRX template to selected or specified clips.
  - Fall back cleanly when Resolve API selection access is insufficient.

- `scripts/collect_batch_context.py`
  - Collect project, timeline, and clip context for preflight and final summary.

## Resolve API Constraints

DaVinci Resolve scripting is suitable for:

- Opening Resolve pages.
- Accessing the current project and timeline.
- Applying DRX grades where supported.
- Reading timeline and clip context.
- Exporting or grabbing stills where supported.
- Applying LUTs to existing graph nodes in supported contexts.

Do not assume the API can reliably:

- Create the full complex node tree from scratch.
- Edit arbitrary primary-wheel parameters by node label.
- Reliably read selected timeline clips in every UI state.
- Decode internal DRX `<Body>` data into editable node details.

When the API is insufficient, Codex should use UI/screen-driven operation rather than hard-coding unstable assumptions.

## Failure Handling

- If DRX template application fails, stop batch processing and report the reason.
- If selected clips cannot be read, ask the user to provide a clear clip range or allow UI-driven selection.
- If the LUT is missing or not discovered by Resolve, stop automatic LUT setup and ask the user to import or refresh LUTs.
- If a clip differs too much from the batch target, apply basic correction and list it for manual review.
- If scopes or image content cannot be read clearly, do not force a judgment; list the clip for manual review.
- If scripting cannot complete a step, fall back to Codex screen operation.

## Verification Plan

Use a small test batch first.

Verify:

- The DRX template imports and applies successfully.
- The node tree order matches the expected structure.
- The LUT node uses the intended LUT.
- The first-frame reference rule is followed.
- RGB Parade is used as the primary scope.
- Visible overexposure is checked after scope-based adjustment.
- Highlight ceiling is lowered when visible overexposure remains.
- Processing does not pause for mid-batch questions.
- The final summary is emitted once.
- No abnormal-condition notes are written into DaVinci node labels, clip markers, or metadata.

## Current Repository Notes

At design time, the local folder contains:

- `davinci调色概述.xmind`
- `assets/templates/Codex_Primary_Batch_Template_v1.drx`
- empty `references/` and `scripts/` directories

The folder is not currently a git repository, so this design cannot be committed until git is initialized or the content is moved into a repository.
