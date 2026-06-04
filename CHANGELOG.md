# Changelog

All notable changes to this project will be documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Initial release with `build` and `restore` composite actions
- `prebuild` input on `build` — runs shell command(s) on cache miss only, before the build (e.g. registry login)
