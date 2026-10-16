---
name: fcpxml-field-notes
description: Use when writing, generating or debugging FCPXML for Final Cut Pro. Rules learned from real imports that Final Cut refused (or quietly broke), most of which the DTD does not catch - frame boundaries, connected-clip offsets, asset uids, timecode, file URLs, transition handles, keyframes, lanes, multicam, keywords, merging projects, import-options.
---

# FCPXML field notes

These rules come from Final Cut Pro rejecting real files written by a generator, or worse, importing them wrong without a word. Each one names the message Final Cut showed, the rule, and how sure we are.

**Passing the DTD is not enough.** Several files below were DTD-valid and still refused, because the rule is about time or media, not grammar. Validate against the DTD *and* check the rules here.

Confidence labels:
- **Measured**: reproduced on a real Final Cut import, fixed, and the fix imported clean.
- **Provisional**: measured on one import; treat as likely, test your own case.

---

## Time and frame boundaries

Error: `The item is not on an edit frame boundary`

### 1. A connected item's offset lives in its parent's timeline
An item on a lane (`lane="1"`, `lane="-1"`, ...) inside a clip is positioned in the **parent's local timeline, which begins at the parent's `start`**, not at zero. Apple's own export:

```xml
<asset-clip ref="r2" offset="0s" start="409/30s" duration="300/30s">
    <asset-clip ref="r3" lane="1" offset="409/30s" duration="60/30s"/>
</asset-clip>
```

Writing `offset="0s"` on that child puts it before its parent's timeline begins, and Final Cut refuses it. A child can never start before the parent's `start`. **Measured.**

### 2. The offset itself must be whole sequence frames
Final Cut checks the connected item's `offset` value, in whole frames of the **sequence**. Being a whole number of the parent clip's frames is not enough. Being whole sequence frames *past the parent's start* is not enough either. Both were tried, and Final Cut still warned. **Measured** on three separate imports.

Exception, **provisional**: when the parent is **retimed** (it has a `<timeMap>`) *and* its frame rate differs from the sequence, its children's offsets count on the **parent's own** frame grid. One import showed this: a logo on a 60 fps clip at 150% speed in a 29.97 timeline was flagged at `8008/30000s` (whole timeline frames) and imported clean at `16/60s` (whole clip frames).

### 3. Quantize once, from the string you wrote
Work out each parent's `start` once, write it, and compute children from **that exact string**. Rounding the same float in two places eventually disagrees by one frame. At 59.94 that is 16 ms, enough for a child to land before its parent and fail the whole sequence. **Measured.**

### 4. Use exact rationals, written the way Apple writes them
- 23.976 → `1001/24000s`, 29.97 → `1001/30000s`, 59.94 → `1001/60000s`. Never `1/30` for 29.97: that drifts one frame every ~33 minutes and lands you back on the frame-boundary error.
- Keep the timebase in the denominator, like Final Cut's own exports: `15/30s`, `15015/30000s`, not a reduced `1/2s`. Reduced values are arithmetically equal. Matching Apple's form makes your files diffable against real exports.

---

## Media, timecode and links

Error: `Invalid edit with no respective media`

### 5. Final Cut merges assets by `uid`
On import, assets with the same `uid` become one asset. If you declare the same file twice, the two get merged and one set of attributes wins. For example, a camera file once as video and once as an audio-only angle. The usual result is a `start` that no longer matches, a source range outside the media, and this error. **Measured.**
- Declare each file **once**. To use a camera's own audio in a multicam, enable it on that camera's video angle (`srcEnable="audio"` or `"all"`) instead of adding a second asset.
- Make the `uid` stable for a given file across re-exports (for example a hash of absolute path + size) and give the `media-rep` the same value as `sig`.

### 6. `start` and `tcFormat` must match the media's own timecode
If the file carries an embedded start timecode (most cameras do) and you declare `start="0s"`, Final Cut can't find the frames you're asking for. Read the timecode from the file (drop-frame is signalled by `;` before the frames field) and declare it. **Measured.**

### 7. Percent-encode the file URL
`<media-rep src>` must be a valid `file://` URL. Spaces and accented characters must be percent-encoded (`My%20Films`, `V%C3%ADdeo.mov`). An unencoded path fails with the "no respective media" error even though the file is right there. Bonus trap: macOS stores accented filenames decomposed (NFD), so normalise before comparing names. **Measured.**

### 8. The declared frame rate must be the file's real rate
Variable-frame-rate footage (GoPro, phones) and true 30 vs 29.97 give Final Cut a different frame count from the one your XML declares, and every clip fails. Probe the real rate; don't assume. **Measured.**

### 9. Source ranges must sit inside the media, in origin-based time
A clip's source range has to fall inside `[asset start, asset start + duration]`. With a `<timeMap>`, the **source** time is each `<timept>`'s `value`; `time` is the output time. Reading `time` as the source passes your own check and fails Final Cut's. **Measured.**

---

## Transitions and fades

Error: `Encountered an unexpected value` (pointing at the `<transition>`)

### 10. A transition is its effect, not its name
Final Cut resolves a transition through `<filter-video ref="...">` pointing at an `<effect>` in `<resources>`. `<transition name="Cross Dissolve"/>` on its own is refused. **Measured.**

### 11. Transitions need real handle frames
A dissolve borrows footage from **before** the incoming clip's in point and **after** the outgoing clip's out point. If those frames don't exist (an incoming clip that starts on the file's first frame, say), the import fails, even though the file is DTD-valid. Trim the transition to the handles both sides can actually give, or drop it and say so. **Measured.**

### 12. The final fade to black ends flush, as the last spine item
Copied from a fade made in Final Cut itself: the transition ends **exactly** at the outgoing clip's end, sits only over that clip, and is the **last element in the spine**, with no gap after it. Adding a gap after it makes it a clip-to-gap transition that needs frames past the clip's end, which fails as in 11. **Measured** against a reference made in Final Cut.

---

## Audio and keyframes

### 13. Keyframes go inside `<keyframeAnimation>`, inside one `<param>`
`<adjust-volume>` only accepts `<param>` children, and a `<param>`'s content model is `fadeIn?, fadeOut?, keyframeAnimation?`. Fades and keyframes both go inside **one** `<param name="amount">`, with the keyframes wrapped in `<keyframeAnimation>`. Bare `<keyframe>` children under `<param>` (a volume duck, for example) are refused. **Measured.**

```xml
<adjust-volume amount="-12dB">
    <param name="amount">
        <fadeIn type="easeIn" duration="30/30s"/>
        <keyframeAnimation>
            <keyframe time="300/30s" value="-12.0"/>
            <keyframe time="308/30s" value="-24.0"/>
        </keyframeAnimation>
    </param>
</adjust-volume>
```
Keyframe times are in the clip's local time, which starts at the clip's `start`.

### 14. Two clips can't overlap on one lane
To crossfade one music track into the next, the outgoing track runs past the join and the two overlap. Put consecutive tracks on alternating lanes (`-1`, `-2`, `-1`, ...). **Measured.**

---

## Structure Final Cut drops without telling you

### 15. Connect only to primary-storyline clips
An item connected to a clip that is itself inside a **connected storyline** (a `<spine lane="...">`) is dropped on import, with no error. Music and logos vanished this way. Anything with children on lanes must sit on the primary storyline. **Measured.**

### 16. `text-style-def` ids are unique across the whole document
They are DTD `ID`s. When you merge several projects into one library and each started its counter at `ts1`, Final Cut refuses the import: `DTD validation failed. (ID ts1 already defined)`. Suffix them per project when merging. **Measured.**

### 17. When merging projects, carry every resource type
Formats, assets, **effects** and multicam **`<media>`** all live in `<resources>`. A merge that forgot `<media>` left an `mc-clip ref` pointing at an id later reused by an audio asset. It imported cleanly and played the wrong thing. Dedupe multicams by their content *after* remapping their inner refs. **Measured.**

### 18. Keywords need both ends of their range
A `<keyword>` (or `<rating>`) with a `start` but no `duration` validates against the DTD, creates its keyword collection, and leaves it **empty**. Put keywords on the outer clip in the timeline (not on a lane copy), after the filters, in DTD child order. Don't "fix" a keyword `start="0s"` on a clip whose own `start` is later: Final Cut writes exactly that itself. **Measured.**

---

## Picture

### 19. `adjust-transform` position is percent of frame *height*
`position="x y"` is in percent of the frame **height**, `+x` right, `+y` up. It isn't pixels, and it isn't percent of width. On 16:9, the quarters of a 2×2 grid sit at `±44.444 ±25`. The Inspector shows pixels, so convert: x px → `x / height × 100`. Rotation is counter-clockwise. **Measured** against a layout built in Final Cut.

---

## Multicam

### 20. `mc-clip` offset and start are two different clocks
`offset` is where the multicam sits in the **sequence**. `start` is where you are in the **multicam's own synced timeline**. They are independent. Use one `<mc-source angleID srcEnable>` per angle, and `srcEnable="all"` when picture and sound come from the same angle. **Measured.**

### 21. A `conform-rate` needs `srcFrameRate`
If you write `<conform-rate>`, include `srcFrameRate` with one of Final Cut's labels (`23.98`, `24`, `25`, `29.97`, `30`, `47.95`, `48`, `50`, `59.94`, `60`, `89.91`, `90`, `100`, `119.88`, `120`). A bare `scaleEnabled`-only element was refused by FCP 11 (`Encountered an unexpected value`). Leave the element out entirely when the clip's rate matches the sequence. **Measured.**

---

## Importing straight into a library

### 22. `import-options` rules
- It must be the **first** child of `<fcpxml>`; elsewhere it is ignored.
- `library location` is a `file:///` URL to the `.fcpbundle` **with a trailing slash**.
- Set `copy assets` explicitly. `0` links the original media; `1` copies it into the library. The default isn't documented.
- Don't also wrap the content in `<library>`. Apple's content model makes them alternatives.

From Apple's content model, and the shape that imports into a library without the chooser.

---

## Check before Final Cut sees it

1. **DTD**: validate against the DTD for the version your file declares. Final Cut ships them inside its app bundle:
   ```sh
   find "/Applications/Final Cut Pro.app" -name "FCPXMLv*.dtd"
   xmllint --noout --dtdvalid /path/to/FCPXMLv1_13.dtd your.fcpxml
   ```
   If the DTD isn't found, fail loudly. A skipped check that reports "verified" is how bare keyframes (13) reached Final Cut.
2. **Rules the DTD can't see**: walk every sequence (not just the first) and check 1, 2, 5, 9, 10, 15, 16 and 18 on the written file.
3. **One real import** for anything new. Most rules on this page were only learned that way.
