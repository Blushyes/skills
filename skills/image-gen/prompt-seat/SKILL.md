---
name: prompt-seat
description: Use when creating, rewriting, auditing, or testing image-generation prompts where visual hierarchy matters. Triggers include prompt engineering for image models, 生图提示词, prompt 排座位, 主位/前排/后排/禁区, NBP-style prompt structure, character posters, character sheets, technical blueprint boards, low-angle posters, and diagnosing why generated images have confused subject/background/atmosphere priority.
---

# Prompt Seat

## Core Idea

Treat an image prompt as a seating chart, not a word pile.

The model may rewrite, expand, or interpret the prompt through its own chain, so never rely on keyword stuffing alone. Make the intended visual priority explicit:

- **Main seat**: the subject and identity baseline.
- **Front row**: the composition controls that must make the image work.
- **Back row**: supporting style, costume, material, light, background, palette, and mood.
- **Exclusions**: known failure modes that should not enter or should not dominate.

Use this as a writing metaphor, not as a claim about a model having a fixed internal seating table.

## Workflow

1. Identify the image type.
   - Character portrait/poster: prioritize face, posture, camera, reading path.
   - Character sheet: prioritize front/side/back views and identity consistency.
   - Product or technical board: prioritize orthographic structure, labels, component callouts.
   - Poster/key visual: prioritize composition center, title-safe areas, visual route.

2. Lock the main seat.
   - Name one subject, not a vague cluster.
   - Include face/identity anchors when the subject is a person or character.
   - Keep style adjectives out of the first sentence unless they define the subject itself.

3. Fill the front row.
   - Specify camera, framing, pose/action, silhouette, and visual reading path.
   - Say what must be seen and what the viewer should notice first/last.
   - For low-angle images, define the closest point and final focus to avoid floor/feet distortion.

4. Move support to the back row.
   - Put costume, material, lighting, palette, environment, and atmosphere after the main structure.
   - Use concrete visual terms before abstract taste words.
   - Keep "cinematic", "premium", "dreamlike", "mysterious", "high-end", and similar terms subordinate.

5. Add exclusions only for real risks.
   - Use exclusions to prevent known failure modes, not to replace clear positive direction.
   - Prefer concrete exclusions: "no fisheye distortion", "no feet dominating foreground", "no plastic skin".
   - Avoid long generic negative lists unless the image type has recurring failures.

6. Shorten after prioritizing.
   - Remove repeated style synonyms.
   - Remove concepts that compete with the main subject.
   - Keep the prompt brief enough that each phrase has a job.

## Prompt Template

```text
Main seat: [single subject], [identity anchors], [face/personality baseline].
Front row: [camera/framing], [pose/action], [composition center], [visual reading path], [must-see structure].
Back row: [costume/material], [environment], [lighting], [palette], [mood as support].
Exclusions: [specific failure modes], [things that must not dominate], no text unless text is required.
```

Delete labels if the target tool responds better to natural prose, but keep the same order and priority.

## Type Micro-Checks

- **Low-angle poster**: name the closest foreground point, then name the final focus. Keep feet/floor from becoming the closest or loudest object unless they are the point of the image.
- **Character sheet**: require same scale, aligned baseline, neutral pose, and consistent face/identity across front, side, and back views.
- **Technical blueprint board**: reserve label margins, keep callouts outside silhouettes, use straight leader lines, and prevent labels from crossing the main forms.

## Rewrite Rules

- If the prompt starts with atmosphere/style, move those words behind subject and composition.
- If many nouns compete for attention, decide which one is the subject and demote the rest.
- If a detail is decorative but loud, either move it to the back row or delete it.
- If the prompt says only "beautiful", "advanced", "high-class", or "cinematic", replace with concrete visual evidence.
- If the user wants a strict visual system, prefer structure over mood: views, axes, spacing, callouts, readable labels.

## Experiment Pattern

When asked to validate the technique with image generation, run a controlled comparison:

- A: pileup prompt with the same elements and weak priority.
- B: ordered prompt with main/front/back seats.
- C: ordered prompt plus targeted exclusions.
- D: style-front prompt that deliberately puts abstract taste words first.

Keep model, seed if available, size, ratio, reference images, and generation settings constant. Evaluate subject lock, visual route, background competition, style overreach, and failure-mode suppression.

Read [references/examples.md](references/examples.md) for a compact experiment example and scoring rubric.
