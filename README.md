# Sovera plugins for Claude Code

The `sovera` plugin lets Claude Code work with a Sovera instance for you: start
tasks, follow runs, answer approval gates, explain failures and find the blueprints
you can use. It talks to Sovera's HTTPS API with `curl`; there is nothing else to
install.

## Install

In Claude Code:

```
/plugin marketplace add wearenotch/sovera-claude-plugins
/plugin install sovera@sovera
```

To match the plugin to your Sovera instance, pin the release tag your operator
gives you. Tags are the release image tags, like `master-<sha8>-<epoch>`:

```
/plugin marketplace add wearenotch/sovera-claude-plugins#<tag>
```

The plugin cannot check this for you: the server does not report which release it
runs, and the plugin's own version number is not tied to a server release.

## Configure

The plugin reads two environment variables. Set them in the shell you start
Claude Code from:

```bash
export SOVERA_URL="https://sovera.example.com"   # your instance, no trailing slash
export SOVERA_API_KEY="vtg_..."                   # your tenant API key
```

Your Sovera operator gives you both. The key is only ever passed to `curl` through
the shell; the plugin is instructed never to print it. Requirements: `curl`, and
`jq` (or `python3`) for building request bodies.

## Use

Ask in plain words, for example:

- "Check my Sovera setup."
- "Run this on Sovera: summarise the attached notes."
- "Start a Sovera run to draft the release notes and tell me when it needs approval."
- "What is my last Sovera run waiting for?"
- "Why did Sovera run 3e86e136-... fail?"
- "Which Sovera blueprints can I use for translation?"

## What it covers

Everything a tenant API key can do: short and long tasks, run status and history,
signal gates, failure explanations, planning rationale and blueprint provenance.
Tenant and key administration is operator work and not part of this plugin.
