# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]
### Added
- Initial release of the LinkedIn n8n Auto-Poster workflow

## [0.1.0] - 2026-06-19
### Added
- Core workflow with manual and schedule triggers (08:00 & 18:00)
- Topic Randomiser with 40 networking/CCNA topics and anti-repeat (last 7)
- Style Randomiser with 8 visual styles and anti-repeat (consecutive)
- Post Maker using DeepSeek v4-flash via OpenRouter for LinkedIn post generation
- Image Prompt Maker using DeepSeek chat via OpenRouter for FLUX prompts
- Image generation pipeline via Cloudflare Workers AI (FLUX.1 Schnell)
- Resilient fallback to text-only post on image generation failure
- Post Trimmer for markdown stripping, hashtag capping (≤3), and length limiting
- LinkedIn publishing (image post primary, text-only fallback)
- Persistent post logging in workflow static data (last 50 runs)
- Slack notifications for run completion/status
- Comprehensive documentation: README, ARCHITECTURE, SETUP, CUSTOMIZATION, SECURITY, TROUBLESHOOTING
- Repository scaffolding: .gitignore, license, contribution guides
