# FCPXML field notes

An agent skill for Claude Code, Codex and other AI coding tools. It holds 22 rules for writing FCPXML that Final Cut Pro actually imports.

Each rule comes from a real import that failed, or one that imported wrong without saying so. Most of them pass the DTD. Each names the error Final Cut showed, the rule, and whether it was measured on a real import.

They were learned the hard way while building the FCPXML exporter for [Cutler](https://cutler.miracontents.com), over more than 550 builds and a lot of refused imports.

## Install

Clone it into your skills folder:

```sh
# Claude Code
git clone https://github.com/erikvideos-tech/fcpxml-field-notes ~/.claude/skills/fcpxml-field-notes
```

For Codex or another tool, clone it wherever that tool reads skills from.

Or just read `SKILL.md`. It's written for people too.

## What's in it

Frame boundaries and connected-clip offsets · asset `uid` merging · timecode and `start` · file URLs · real frame rates · source ranges and `timeMap` · transition effects and handles · the final fade · keyframes · lanes · items Final Cut drops silently · merging projects · keywords · `adjust-transform` units · multicam clocks · `conform-rate` · `import-options` · how to check a file before Final Cut does.

## Contributing

Found another rule the hard way? Open an issue or a pull request with the error Final Cut showed and the smallest file that reproduces it.

## Licence

MIT
