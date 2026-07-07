# Privacy Policy

**Last updated:** 2026-07-07

## What this is

claude-helm is an open source plugin for Claude Code. It runs entirely on your local machine. There is no server, no account system, and no data collection.

## Data we collect

We collect no data. claude-helm does not transmit any information to us, to third parties, or to any remote service.

Specifically, we do not collect:

- Personal information (name, email, IP address)
- Usage data or analytics
- Project files, code, or CLAUDE.md content
- Crash reports or error logs

## How the plugin works

claude-helm consists of markdown instruction files that are read by Claude Code on your machine. All processing happens locally. Any files the plugin creates (CLAUDE.md, README.md, legal documents, commit messages) are written to your local project directory. Nothing leaves your machine via this plugin.

## Third-party tools

Some commands instruct Claude Code to use third-party tools you have independently installed:

- **Git** — for version control operations
- **GitHub CLI (`gh`)** — for creating GitHub Releases, if you choose to use that feature

These tools operate under their own terms and privacy policies. claude-helm does not control or access any data they handle.

## Claude Code and Anthropic

claude-helm runs inside Claude Code, which is a product of Anthropic. When Claude Code processes the plugin's instructions, your prompts and project context are sent to Anthropic's API according to [Anthropic's privacy policy](https://www.anthropic.com/privacy). This is a function of Claude Code, not of claude-helm.

## Your rights under GDPR

Because we collect no personal data, there is no data held about you for us to access, correct, delete, or transfer. If you have any questions, contact us at the address below.

## Contact

For questions about this Privacy Policy, open an issue at [github.com/myowinthein/claude-helm/issues](https://github.com/myowinthein/claude-helm/issues).
