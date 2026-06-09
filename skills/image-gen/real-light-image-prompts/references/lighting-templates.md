# Lighting Templates Reference

Use this reference when the user asks for full templates, example prompts, stability tests, or experiment design.

## Contents

- Tested Conditions
- Template A: Practical Overhead Fluorescent Flatness
- Template B: Blown-Window Auto Exposure
- One-Line Prompt Compression
- Scene Adaptation Rules
- Failure Diagnosis
- Stability Test Rubric

## Tested Conditions

The templates were derived from prompt-only image experiments targeting real-photo feel:

- Model family: GPT Image 2 style prompt-only image generation
- Quality: 2k
- Reference policy: no reference image
- Goal: transfer real, ordinary, phone-origin lighting across unrelated scenes
- Stable threshold: average score 4 or above across three scenes

The method is model-agnostic: keep the lighting mechanism, camera imperfection, material evidence, and exclusions even when using another image model.

## Template A: Practical Overhead Fluorescent Flatness

Use this when realism should come from cheap indoor practical lighting rather than attractive light.

Best scenes:

- Neighborhood restaurants and cafeterias
- Clinics, waiting rooms, corridors, classrooms
- Convenience stores, laundromats, offices
- Workshops, small factories, repair shops

Core template:

```text
Main seat: [ordinary real-life indoor scene], photographed as a casual phone snapshot, not aesthetic photography.
Front row: [4:3 or target-ratio human-height crop], accidental framing, foreground clutter or partial occlusion, one small human action anchors the eye, background context remains visible.
Back row: visible overhead fluorescent tubes or cheap ceiling fixtures, flat top light, weak soft shadows under faces and objects, gray-green phone color cast, modest noise and JPEG compression, worn surfaces and everyday mess.
Exclusions: no cinematic lighting, no window-beauty light, no warm commercial ambience, no luxury interior, no staged stock photo, no perfect readable signs, no retouched faces, no illustration, no render.
```

Example prompt, noodle restaurant:

```text
Main seat: small neighborhood noodle restaurant at night, photographed as a casual phone snapshot, not aesthetic photography.
Front row: 4:3 human-height crop from a customer standing between tables, bowls, chopsticks and plastic baskets clutter the foreground, a cook wipes a counter while customers eat in the middle distance, accidental framing with people partly cut by edges.
Back row: visible overhead fluorescent tubes and cheap ceiling fixtures, flat top light, weak soft shadows under faces and objects, gray-green phone color cast, modest noise and JPEG compression, worn tile walls, laminated menu signs with unreadable text, greasy everyday mess.
Exclusions: no cinematic lighting, no window-beauty light, no warm commercial ambience, no luxury interior, no staged stock photo, no perfect readable signs, no retouched faces, no illustration, no render.
```

Example prompt, clinic:

```text
Main seat: community clinic waiting room, photographed as a casual phone snapshot, not aesthetic photography.
Front row: 4:3 human-height crop from the back of the room, plastic chairs and a water dispenser partially block the foreground, several patients wait with tired postures, one nurse passes through a doorway, background context remains visible.
Back row: visible fluorescent tubes in a low ceiling, flat top light, weak soft shadows under faces and chairs, gray-green phone color cast, modest noise and JPEG compression, old white walls, worn posters with unreadable medical text, everyday clutter.
Exclusions: no cinematic lighting, no warm commercial ambience, no luxury hospital, no staged stock photo, no perfect readable signs, no retouched faces, no illustration, no render.
```

## Template B: Blown-Window Auto Exposure

Use this when realism should come from phone auto-exposure struggling with a bright window or open door.

Best scenes:

- Apartment kitchens and living rooms
- Small shops beside open doors
- Buses, taxis, trains, waiting areas
- Workshops, tailor shops, back rooms, offices near windows

Core template:

```text
Main seat: [ordinary real-life scene near a bright window or open door], photographed as a casual phone snapshot, not aesthetic photography.
Front row: [4:3 or target-ratio human-height crop], bright window or door is visible at one side or behind the subject, people or objects partly silhouette against it, imperfect crop and casual body positions.
Back row: phone auto-exposure protects some midtones but clips the window, soft glare, low local contrast near the light, muted indoor color, modest noise and JPEG compression, real clutter and worn surfaces.
Exclusions: no professional backlight, no cinematic rim light, no golden-hour beauty look, no HDR recovery, no perfect bokeh, no luxury interior, no retouched faces, no illustration, no render.
```

Example prompt, apartment kitchen:

```text
Main seat: small apartment kitchen in the morning, photographed as a casual phone snapshot near a bright window, not aesthetic photography.
Front row: 4:3 human-height crop from the doorway, a messy table with bowls and vegetables fills the foreground, one person cooks by the counter and is partly silhouetted by the window, imperfect crop and casual posture.
Back row: bright window on the left is clipped white, phone auto-exposure keeps the room dim but readable, soft glare, low local contrast near the window, muted indoor color, modest noise and JPEG compression, old cabinets and real kitchen clutter.
Exclusions: no professional backlight, no cinematic rim light, no golden-hour beauty look, no HDR recovery, no perfect bokeh, no luxury kitchen, no retouched faces, no illustration, no render.
```

Example prompt, city bus:

```text
Main seat: city bus interior with passengers near a bright side window, photographed as a casual phone snapshot, not aesthetic photography.
Front row: 4:3 seated passenger perspective, seat backs and a handrail partly block the foreground, a few passengers sit or stand casually, faces are slightly underexposed against the window, imperfect crop and mild motion softness.
Back row: large side windows are clipped white with daylight glare, phone auto-exposure keeps the aisle dim but readable, soft flare on glass, muted interior color, modest noise and JPEG compression, worn seats, ordinary transit clutter.
Exclusions: no professional backlight, no cinematic rim light, no travel advertisement look, no HDR recovery, no perfect bokeh, no luxury vehicle, no retouched faces, no illustration, no render.
```

## One-Line Prompt Compression

When the user asks for one line, preserve the order: scene, phone camera, crop, light mechanism, material evidence, exclusions.

Overhead fluorescent one-liner:

```text
[Scene], casual phone snapshot from [viewpoint], accidental [ratio] crop with foreground clutter and a small human action, visible overhead fluorescent tubes or cheap ceiling fixtures, flat top light, weak soft shadows, gray-green phone color cast, worn surfaces, modest noise and JPEG compression, no cinematic lighting, no warm commercial ambience, no staged stock photo, no perfect readable signs, no retouching, no render.
```

Blown-window one-liner:

```text
[Scene] near a bright window or open door, casual phone snapshot from [viewpoint], imperfect [ratio] crop with foreground clutter and casual posture, window or doorway clipped white, people or objects partly silhouetted, phone auto-exposure leaves the room dim but readable, muted indoor color, modest noise and JPEG compression, no professional backlight, no golden-hour beauty, no HDR recovery, no perfect bokeh, no retouching, no render.
```

For 21:9:

```text
Use "wide phone panorama or accidental horizontal crop" and exclude "cinematic widescreen, anamorphic lens, dramatic movie-still composition, perfect color grading".
```

## Scene Adaptation Rules

- Replace generic "realistic" with visible causes: tubes, fixtures, clipped window, glare, weak shadows, compression.
- Add one mundane human action: waiting, eating, wiping, restocking, cooking, sewing, checking a phone.
- Keep signage partial or unreadable unless the user needs exact text.
- Use clutter that belongs to the scene, not random mess.
- Keep faces secondary unless the user specifically needs a portrait.
- Keep ratio separate from style. A 21:9 image can still be a phone panorama rather than a film still.

## Failure Diagnosis

| Failure | Likely Cause | Fix |
| --- | --- | --- |
| Cinematic or editorial look | The prompt front-loaded aesthetic words or lens language | Move style to exclusions; add visible practical light or exposure failure |
| Too clean or expensive | The environment lacks wear and clutter | Add old surfaces, cheap fixtures, plastic objects, partial occlusion, everyday mess |
| Generic brightness | The light source is not visible or exposure behavior is vague | Name the visible fixture/window and what happens to shadows/highlights |
| Stock-photo staging | People are posed and the frame is too balanced | Add accidental crop, edge cutoffs, foreground blockage, task behavior |
| AI/rendered feel | Surface and camera artifacts are missing | Add modest noise, JPEG compression, imperfect faces, uneven light; exclude render/CGI |
| Wide ratio becomes movie still | 21:9 implies cinema to the model | Say wide phone panorama or accidental horizontal crop; exclude cinematic widescreen/anamorphic |

## Stability Test Rubric

Generate at least three unrelated scenes per template while holding model, quality, ratio, and reference policy constant.

Score each output from 1 to 5:

- Light transfer: the lighting mechanism remains visible across scenes.
- Realism: the output feels like a casual phone-origin photo.
- Anti-polish: it avoids cinematic, commercial, HDR, and stock-photo habits.
- Scene obedience: the requested place and action remain clear.
- Exposure behavior: fluorescent flatness or clipped-window behavior appears as specified.

Average 4 or above means the template is stable enough to reuse. If one scene fails, rewrite the scene-specific front row before changing the lighting mechanism.
