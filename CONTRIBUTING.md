# Contributing to dns-deep-dive

Contributions welcome! This repo aims to be the most complete, accurate, low-level DNS reference on GitHub.

## What We Need

- 🐛 **Corrections** — factual errors, outdated information
- 📝 **New topics** — missing record types, edge cases, newer RFCs
- 🧪 **New labs** — hands-on exercises, Docker-based labs
- 📊 **Diagrams** — ASCII or image-based architecture diagrams
- 📦 **Packet captures** — real `.pcap` files with annotations
- 🌍 **Translations** — content in other languages

## How to Contribute

1. Fork the repository
2. Create a feature branch: `git checkout -b add-topic-X`
3. Make your changes
4. Verify any zone files: `named-checkzone <origin> <file>`
5. Submit a pull request

## Style Guide

### Markdown

- Use `##` for main sections, `###` for subsections
- Code blocks with language hints: ` ```bash ` or ` ```
- Tables for comparisons
- Navigation links at bottom of each doc file

### Zone Files

- Include `$ORIGIN` and `$TTL` directives
- Comment every record type with a brief explanation
- Use RFC-compliant example IPs: `192.0.2.x` (TEST-NET-1), `2001:db8::x`
- Never use real public IPs unless for illustration with attribution

### Code Examples

- Test all commands before submitting
- Include expected output where helpful
- Note OS-specific variations (Linux vs macOS vs Windows)

## Reporting Issues

Use GitHub Issues with the appropriate template:
- **Incorrect information** — factual errors
- **Missing topic** — something not covered

## RFC References

When citing RFCs:
- Link to https://datatracker.ietf.org/doc/html/rfcXXXX
- Note if RFC is obsoleted or has errata

## Code of Conduct

Be respectful and constructive. This is a technical resource — focus on accuracy and clarity.
