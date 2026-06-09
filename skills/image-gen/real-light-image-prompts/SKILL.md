---
name: real-light-image-prompts
description: Create, rewrite, audit, and test image-generation prompts for realistic photo lighting and phone-snapshot realism. Use for prompt-only image generation when users want real-photo style, 真实照片画风, practical overhead fluorescent light, blown-window auto exposure, low-cost interior realism, anti-cinematic lighting, or diagnosis of images that look too polished, stock, HDR, bokeh-heavy, staged, or AI-rendered.
---

# Real Light Image Prompts

## Core Idea

Real-photo style comes from a believable lighting mechanism, not from stacking words like "realistic" or "cinematic".

Anchor the prompt in ordinary physical evidence: visible light source, imperfect exposure, casual phone framing, worn materials, clutter, modest noise, and targeted exclusions against polished image habits.

## Workflow

1. Choose the scene before the style.
   - Name an ordinary place, action, and time condition.
   - Prefer lived-in, low-design environments over iconic or aspirational sets.
   - Keep the scene specific enough to obey, but not so branded or text-heavy that signs become the subject.

2. Choose one lighting mechanism.
   - Use **practical overhead fluorescent flatness** for cheap indoor public or semi-public places: restaurants, clinics, stores, offices, classrooms, workshops, corridors.
   - Use **blown-window auto exposure** for daylight interiors, vehicles, small shops, kitchens, and workspaces with a bright window or open door in frame.
   - Do not combine both mechanisms unless the user asks for a messy mixed-light environment; one strong mechanism transfers more reliably.

3. Seat the prompt in this order.
   - **Main seat**: the ordinary scene and action, photographed as a casual phone snapshot.
   - **Front row**: crop, viewpoint, framing accident, foreground occlusion, small human action, and what remains visible.
   - **Back row**: visible light source, exposure behavior, color cast, surface wear, clutter, compression/noise.
   - **Exclusions**: concrete polish risks: cinematic light, commercial staging, HDR, bokeh, luxury interior, retouching, render/illustration.

4. Keep camera language phone-native.
   - Say "casual phone snapshot", "human-height crop", "standing customer perspective", "doorway view", or "seated passenger perspective".
   - Avoid film and lens prestige terms unless the user wants a deliberately photographic look.
   - For 21:9 or other wide ratios, add "wide phone panorama or accidental horizontal crop" and exclude cinematic widescreen/anamorphic framing.

5. Add realism evidence without overexplaining.
   - Use visible fixtures, clipped windows, weak shadows, gray-green or muted indoor color, worn surfaces, foreground clutter, partial occlusion, unreadable signage, modest JPEG compression.
   - For people, prefer casual posture and task behavior over perfect faces, beauty poses, or fashion-model language.

6. Output in the format the user needs.
   - If the user asks for a one-line prompt, compress the selected template into one sentence while preserving the same priority order.
   - If the user asks to run experiments, read `references/lighting-templates.md` and test each mechanism across at least three unrelated scenes.
   - If the user provides a failed prompt or image result, diagnose which mechanism got lost before rewriting.

## Compact Templates

Use these when the user wants a fast rewrite. Read `references/lighting-templates.md` for full tested versions, examples, and scoring.

### Practical Overhead Fluorescent Flatness

```text
Main seat: [ordinary real-life indoor scene], photographed as a casual phone snapshot, not aesthetic photography.
Front row: [phone viewpoint and crop], accidental framing, foreground clutter or partial occlusion, one small human action anchors the eye, background context remains visible.
Back row: visible overhead fluorescent tubes or cheap ceiling fixtures, flat top light, weak soft shadows, gray-green phone color cast, modest noise and JPEG compression, worn surfaces and everyday mess.
Exclusions: no cinematic lighting, no window-beauty light, no warm commercial ambience, no luxury interior, no staged stock photo, no perfect readable signs, no retouched faces, no illustration, no render.
```

### Blown-Window Auto Exposure

```text
Main seat: [ordinary real-life scene near a bright window or open door], photographed as a casual phone snapshot, not aesthetic photography.
Front row: [phone viewpoint and crop], bright window or door visible at one side or behind the subject, people or objects partly silhouetted, imperfect crop and casual body positions.
Back row: phone auto-exposure protects some midtones but clips the window, soft glare, low local contrast near the light, muted indoor color, modest noise and JPEG compression, real clutter and worn surfaces.
Exclusions: no professional backlight, no cinematic rim light, no golden-hour beauty look, no HDR recovery, no perfect bokeh, no luxury interior, no retouched faces, no illustration, no render.
```

## Diagnosis Rules

- If the image looks cinematic, remove film/lens glamour and add a visible practical light source or phone exposure failure.
- If the image looks like stock photography, add accidental crop, partial occlusion, clutter, worn surfaces, and non-posed human actions.
- If the image looks too clean or expensive, specify cheap fixtures, old walls, worn counters, plastic chairs, price tags, tools, cables, stains, or everyday mess appropriate to the scene.
- If the image looks like a render, add compression, mild noise, imperfect faces, uneven light, and exclude illustration/render/CGI.
- If a wide image turns into a movie still, say "wide phone panorama or accidental horizontal crop" and exclude cinematic widescreen, anamorphic lens, dramatic composition, and perfect color grading.

## Experiment Pattern

When validating a template, hold model, size, ratio, quality, and reference policy constant. Generate at least three unrelated scenes per lighting mechanism, then score:

- Light transfer: the same lighting mechanism remains visible across scenes.
- Realism: the image reads as a casual phone-origin photo.
- Anti-polish: it avoids cinematic, commercial, and stock-photo aesthetics.
- Scene obedience: the requested place and action are clear.
- Exposure behavior: the light behaves as specified instead of becoming generic brightness.

Treat an average score of 4 or above across scenes as stable enough to reuse.
