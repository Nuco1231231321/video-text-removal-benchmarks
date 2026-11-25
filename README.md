# Removing burned-in text from video: notes and benchmarks

Field notes for the specific case where text is **already encoded into the frames** — hardcoded
subtitles, open captions, TikTok caption text, lower thirds, handles, logos. Not subtitle tracks.
Not overlays. Pixels.

The short version: most advice on the internet answers "remove subtitles" with a command that only
works when the subtitles are a separate stream. This repository collects the commands, the
tradeoffs, and the cases where each method quietly fails.

Maintained by the team behind [CleanVideoText](https://cleanvideotext.com/), a browser tool in this
space. The ffmpeg recipes below are tool-agnostic and belong to nobody.

## 1. Decide which problem you have

A container can carry subtitles in two ways, and they need completely different fixes.

| | Soft subtitles | Hard subtitles (open captions) |
|---|---|---|
| Where the text lives | A separate stream | The video frames |
| Player can turn it off | Yes | No |
| Fix | Drop the stream | Repair pixels |
| Re-encode video | Not needed | Required |

Detect before doing anything else:

```bash
ffprobe -v error -show_entries stream=index,codec_type,codec_name -of csv=p=0 input.mp4
```

A line containing `subtitle` means there is a subtitle stream. If there is no such line and the text
is still visible with subtitles switched off, the text is burned in.

## 2. Soft subtitles: remove the stream, not the quality

```bash
ffmpeg -i input.mp4 -map 0 -map -0:s -c copy output.mp4
```

`-map 0` keeps everything, `-map -0:s` drops subtitle streams, `-c copy` avoids re-encoding. This is
lossless and takes seconds. If a player still shows text after this, the text was never a stream.

## 3. Hard subtitles: four routes, four different failures

| Method | Removes text | Keeps framing | Keeps background | Moving text |
|---|---|---|---|---|
| Crop | Deletes the region | No | No | Sometimes |
| Cover / blur | Hides it | Yes | No | With tracking |
| `delogo` | Interpolates from edges | Yes | Approximate | No |
| AI inpainting | Rebuilds from nearby frames | Yes | Best available | With a good mask |
| Re-export from project | Yes | Yes | Yes | Yes |

**Crop** — cheapest and most destructive to composition:

```bash
ffmpeg -i input.mp4 -vf "crop=iw:ih-120:0:0" -c:v libx264 -crf 18 -c:a copy output.mp4
```

**Blur band** — hides rather than removes, and on motion it muddies the frame under the bar:

```bash
ffmpeg -i input.mp4 -filter_complex \
  "[0:v]crop=iw:120:0:ih-120,boxblur=20:5[band];[0:v][band]overlay=0:H-120[out]" \
  -map "[out]" -map 0:a? -c:v libx264 -crf 18 -c:a copy output.mp4
```

**`delogo`** — no temporal model, so it smears on detailed or moving backgrounds:

```bash
ffmpeg -i input.mp4 -vf "delogo=x=120:y=760:w=840:h=80" -c:v libx264 -crf 18 -c:a copy output.mp4
```

**Inpainting** — the only route that keeps framing *and* reconstructs the background, at the cost of
a mask. Open-source starting points: [ProPainter](https://github.com/sczhou/ProPainter),
[E2FGVI](https://github.com/MCG-NKU/E2FGVI). Both need a per-frame mask; producing that mask is
usually the actual work.

## 4. What none of these can do

If the text covered a face, a hand, or a moving object, there is no background to restore. The model
invents plausible pixels. There is no command that recovers information the camera never captured.
Review the first and last frame of every cue before shipping.

## 5. Verify before you commit to a full render

Run the worst five seconds only, then pull frames at the cue boundaries:

```bash
ffmpeg -ss 00:00:12.300 -i output.mp4 -frames:v 1 frame-start.png
ffmpeg -ss 00:00:14.800 -i output.mp4 -frames:v 1 frame-end.png
```

Look for ghost letters, a soft rectangle around the repair, smearing that follows camera motion, a
pop at the cue boundary, and audio drift after re-encoding.

## 6. Deeper guides

Scenario-specific walkthroughs, including where each method breaks:

- [Remove subtitles from a video without blurring or cropping](https://cleanvideotext.com/remove-subtitles-from-video)
- [Remove captions from a video](https://cleanvideotext.com/remove-captions-from-video)
- [Remove text from a TikTok video](https://cleanvideotext.com/remove-text-from-tiktok-video)
- [Remove an object from a video](https://cleanvideotext.com/remove-object-from-video)
- [Remove a watermark from video footage you own](https://cleanvideotext.com/remove-watermark-from-video)

## 7. Rights

Only edit footage you own or are licensed to edit. Removing a watermark from someone else's video is
a rights problem, not a technical one. Every command here is deliberately general; the
responsibility stays with the editor.

## Contributing

Corrections and measured comparisons are welcome — especially cases where `delogo` beats a heavier
model, or where inpainting visibly fails. Please include the input characteristics (resolution,
motion, background detail) with any result.
