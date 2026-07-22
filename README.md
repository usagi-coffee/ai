# ai

This repository contains AI-related things, including reusable skills and other agent-facing artifacts.

## Skills

- [android-aarch64](./.agents/android-aarch64/SKILL.md): build and validate Android projects with x86_64 SDK, NDK, and Gradle tools on ARM Linux
- [svelte](./.agents/svelte/SKILL.md): create and edit Svelte 5 and SvelteKit code with concise patterns for runes, reactivity, bindings, attachments, and domain models
- [tree-sitter-optimize](./.agents/tree-sitter-optimize/SKILL.md): guidance for running measured tree-sitter grammar optimization passes

This repo can be used with [skills.sh](https://skills.sh/). Install the skill with:

```sh
npx skills add https://github.com/usagi-coffee/ai --skill android-aarch64
npx skills add https://github.com/usagi-coffee/ai --skill svelte
npx skills add https://github.com/usagi-coffee/ai --skill tree-sitter-optimize
```
