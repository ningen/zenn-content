# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Zenn content repository for publishing technical articles and books. Zenn is a Japanese platform for developers to share technical content. The repository uses the Zenn CLI to manage and preview content.

## Common Commands

### Creating New Content
```bash
# Create a new article with auto-generated slug based on current branch and date
npm run new:article

# The slug format is: {branch-name}-{yy}y{mm}m{dd}d
```

### Preview Content
```bash
# Start local preview server to view articles and books
npm run preview
```

### Testing
```bash
# No specific test command configured
npm test  # Will output error message
```

## Repository Structure

- `articles/` - Markdown files for individual articles
  - Each article has frontmatter with title, emoji, type, topics, and published status
  - Articles are written in Japanese
  - Naming convention: descriptive-slug-{yy}y{mm}m{dd}d.md

- `books/` - Directory for book content (currently empty)

- `scraps/` - JSON files for Zenn scraps (short notes/snippets)

- `images/` - Static images organized by article slug subdirectories

## Content Guidelines

### Article Frontmatter Format
```yaml
---
title: "Article Title in Japanese"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア  
topics: ["AI", "MCP"]
published: false
---
```

### File Naming
- Articles use descriptive slugs with date suffixes
- Images are organized in subdirectories matching article slugs
- Branch-based naming for new articles via npm script

## Development Workflow

1. Create new branch for article
2. Run `npm run new:article` to generate article file with auto-slug
3. Write content in Japanese
4. Use `npm run preview` to review locally
5. Set `published: true` when ready to publish
6. Commit and push changes

## Dependencies

- `zenn-cli` (v0.1.140) - Official Zenn command line interface for content management and preview