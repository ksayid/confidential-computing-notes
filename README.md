# Confidential Computing Notes

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![GitHub Pages](https://img.shields.io/badge/docs-View%20Site-blue)](https://ksayid.github.io/confidential-computing-notes/)
[![GitHub Repo stars](https://img.shields.io/github/stars/ksayid/confidential-computing-notes?style=social)](https://github.com/ksayid/confidential-computing-notes/stargazers)
[![GitHub forks](https://img.shields.io/github/forks/ksayid/confidential-computing-notes?style=social)](https://github.com/ksayid/confidential-computing-notes/network)
[![Contributions Welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg)](https://github.com/ksayid/confidential-computing-notes/issues)

A structured, open-source knowledge base on confidential computing — covering trusted execution environments, attestation, Intel TDX/SGX, confidential containers, and cloud security.

## Why This Exists

Confidential computing is a fast-moving field spread across vendor docs, academic papers, and specification PDFs. This site distills those sources into a single, searchable reference organized by topic — so you can understand how TEEs, attestation, and confidential containers actually work without piecing it together yourself.

> [!NOTE]
> All opinions and explanations in this repo are my own and do not necessarily reflect the official stance of my employer.

## What's Covered

### [Core Concepts](https://ksayid.github.io/confidential-computing-notes/docs/core/)
Foundational topics including confidential computing principles, TEE security models, Intel SGX enclaves, Intel TDX (overview, architecture, ABI, trusted I/O), encrypted filesystems, secure key release, paravisors, and live migration of confidential VMs.

### [Container Technologies](https://ksayid.github.io/confidential-computing-notes/docs/containers/)
Container fundamentals, Kata Containers, Confidential Containers (CoCo), and Kubernetes integration for running workloads inside TEEs.

### [Tools & Frameworks](https://ksayid.github.io/confidential-computing-notes/docs/tools/)
gRPC and Open Policy Agent as they relate to confidential computing infrastructure.

### [Reference](https://ksayid.github.io/confidential-computing-notes/docs/misc/)
Cloud fundamentals and supplementary notes.

## Local Development

The site is built with [Jekyll](https://jekyllrb.com/) using the [just-the-docs](https://just-the-docs.com/) theme and hosted on GitHub Pages.

### Prerequisites

- Ruby (with `gem`)
- [Bundler](https://bundler.io/) (`gem install bundler`)

### Setup

```bash
git clone git@github.com:ksayid/confidential-computing-notes.git
cd confidential-computing-notes
bundle install
```

### Run locally

```bash
bundle exec jekyll serve
```

Then open [http://localhost:4000](http://localhost:4000).

## Repository Structure

```
docs/
├── core/           # TEEs, SGX, TDX, attestation, key release, live migration
├── containers/     # Kata, CoCo, Kubernetes, container fundamentals
├── tools/          # gRPC, Open Policy Agent
├── misc/           # Cloud fundamentals, scratch notes
└── img/            # Architecture diagrams
_config.yml         # Jekyll configuration (theme, search, navigation)
_sass/              # Custom style overrides
index.md            # Site homepage
```

## Contributing

Contributions are welcome — whether it's fixing a typo, improving an explanation, or adding a new topic.

1. Fork the repository
2. Create a branch (`git checkout -b my-topic`)
3. Add or edit Markdown files under `docs/` in the appropriate section
4. Test locally with `bundle exec jekyll serve`
5. Open a pull request

Each content page uses [just-the-docs front matter](https://just-the-docs.com/docs/navigation-structure/) for navigation:

```yaml
---
title: Page Title
parent: Core Concepts    # must match the parent index title
nav_order: 5
---
```

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.
