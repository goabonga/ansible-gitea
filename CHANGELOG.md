# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

Entries are generated from [Conventional Commits](https://www.conventionalcommits.org/)
by [multicz](https://github.com/goabonga/multicz); do not edit them by hand.

## [0.1.0] - 2026-07-30

### Added

- **gitea**: add the gitea guest with postgresql and organisations (`9a59298`)
- **act_runner**: add the gitea actions agents (`3e69910`)
- **internal_dns**: serve the lab domain with dnsmasq (`bde4faf`)
- **internal_resolver**: route the lab domain to the internal resolver (`d7aa215`)
- **step_ca**: serve gitea over tls with an internal certificate authority (`3e2fafb`)
