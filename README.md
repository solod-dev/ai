# Solod AI plugin

A Claude Code plugin to write programs in [Solod](https://github.com/solod-dev/solod) (So) — a subset of Go that transpiles to C.

## Installation

In Claude Code, run:

```
/plugin marketplace add solod-dev/ai
/plugin install solod@solod-plugins
```

Then approve the plugin when Claude Code prompts you.

## Usage

Once installed, Claude Code automatically uses the skill when you ask it to write or modify Solod code. No special commands needed — just describe what you want:

> Write "hello world" in Solod.

> Write a Solod program that reads a file line by line and prints each line in reverse.

> Convert this Go program to Solod.
