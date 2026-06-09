# Prompt Seat Examples

Load this file when you need examples, an A/B experiment setup, or an evaluation rubric.

## Example: Ancient Swordswoman Poster

Control settings used in the source experiment:

- Model: GPT Image 2
- Quality: 2k
- Ratio: 3:4
- Constant subject: ancient Chinese female swordswoman in rain, temple courtyard, muted jade/black/gold palette

### A. Pileup

```text
Dreamlike, cinematic, premium, mysterious, ethereal, ornate, high-end fantasy atmosphere, glossy dramatic light, complex palace background, rain, floating silk, smoke, jade, gold, sword, embroidery, phoenix motifs, ancient Chinese female swordswoman, beautiful face, full costume, low angle, heroic poster, detailed fabric, wet stone floor, dramatic clouds, elegant mood, no text.
```

Expected behavior: attractive and detailed, but background, props, costume, and atmosphere may compete with the face.

### B. Ordered

```text
Main seat: one ancient Chinese female swordswoman, original character, clear memorable face, calm determined eyes, half-body portrait. Front row: low-angle poster composition from waist level, face is the final visual focus, sword held close to the body, clean silhouette, reading path from waist and sword upward to the face. Back row: dark temple courtyard in light rain, muted jade and black costume, restrained gold embroidery, wet stone reflections, soft cinematic side light, quiet mist, no text.
```

Expected behavior: stronger subject lock and clearer reading path; less decorative spectacle.

### C. Ordered With Exclusions

```text
Main seat: one ancient Chinese female swordswoman, original character, clear memorable face, calm determined eyes, half-body portrait. Front row: low-angle poster composition from waist level, face is the final visual focus, sword held close to the body, clean silhouette, reading path from waist and sword upward to the face. Back row: dark temple courtyard in light rain, muted jade and black costume, restrained gold embroidery, wet stone reflections, soft cinematic side light, quiet mist. Exclusions: no fisheye distortion, no feet or floor dominating the foreground, no plastic skin, no tourist photo feeling, no cheap cosplay feeling, no overdecorated background, no text.
```

Expected behavior: most controlled; targeted failures are suppressed, sometimes at the cost of drama.

### D. Style Front

```text
Premium cinematic atmosphere first, mysterious ethereal high-end visual impact, dreamlike ancient fantasy mood, dramatic rain and glowing mist, luxurious gold-and-jade texture, epic poster feeling, stylish low-angle visual, beautiful ancient Chinese female swordswoman somewhere in the scene, ornate costume, temple courtyard, sword, no text.
```

Expected behavior: strong atmosphere but weaker subject authority; environment and style may become the real task.

## Evaluation Rubric

Score each output from 1 to 5:

- Subject lock: Is the intended subject unmistakably primary?
- Face/identity clarity: Is the face or identity anchor readable and memorable?
- Visual path: Does the eye travel through the intended route?
- Support discipline: Do costume, background, and mood support rather than compete?
- Failure suppression: Did exclusions prevent concrete known issues?
- Prompt economy: Did each phrase have a clear job?

Useful conclusion phrasing:

```text
The result supports the seating-chart hypothesis if the ordered prompt improves hierarchy and reading path without changing the topic or generation settings. The result does not prove that word order is a universal control mechanism; it shows that explicit priority structure can steer how the prompt is interpreted.
```
