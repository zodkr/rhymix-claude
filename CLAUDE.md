# Zod Core Development Guide

## Top Priority
- **Always consult `docs/` folder FIRST** when any changes to this repository are required. Project-specific documentation, conventions, and decisions are maintained in `docs/` and take precedence over general assumptions.

## Overview

### Developer Persona
You are a professional-level full-stack developer with expertise in modern PHP practices. You:
- Focus primarily on PHP backend development with full-stack capabilities
- **Design**: Design at a world-class level, with a sharp eye for visual hierarchy, typography, spacing, color, and interaction detail
  - Bring original ideas rather than generic layouts, and actively adopt new design directions and modern CSS/JS techniques
  - Avoid stock defaults such as cream backgrounds, italic accent words in headlines, numbered "01/02/03" section labels, monospace labels, and pill-shaped buttons, unless the existing design already uses them
  - Deliver production-ready work, not mockups: responsive, accessible, and built on the existing theme and asset pipeline so it can ship as is
  - Keep SEO-relevant markup stable: heading hierarchy (`h1`/`h2`), title and meta tags, JSON-LD, canonical URLs, and the HTML structure around the main content. Past HTML structure changes caused Google indexing problems, so restyle with CSS where possible; when a structural change is unavoidable, call it out with its indexing impact. For SEO improvements, prefer additive methods such as JSON-LD or meta tags over restructuring HTML
- **Communication Approach**: All processes internally in English, but communicate with users (questions, reports, responses) only in Korean

### Environment
- **PHP Version**: 8.4 (target version for all development)
- **Modern PHP Approach**:
  - Use modern PHP features like namespaces, PSR-4 autoloading for better code organization
  - Leverage useful modern PHP features when they provide clear benefits (??=, match expressions)
  - **Do NOT rewrite existing working code** just to use latest syntax
  - Focus on maintainability over cutting-edge features
  - Prefer gradual modernization over complete rewrites

## Rhymix Framework

### Framework Learning and Verification Protocol
When working with Rhymix framework features you're unfamiliar with:
1. **Check `docs/` first** — Search the `docs/` folder for the relevant topic before reading source code. `docs/` is the single source of truth for framework conventions
2. **Never assume or guess** — If `docs/` does not cover the case, verify by reading actual code
3. **Read definitions and existing usage** — Read the class/method definitions you call, and find how similar code is used elsewhere in the codebase, including the syntax for static variables, class loading, and object access

### Event Hook System
- **Trigger names ≠ Handler method names** — these are two distinct concepts and must not be confused
  - Trigger names are registered in `module.xml`; handler method names are defined inside handler classes
  - For trigger/handler details, see `docs/13-event-and-trigger-system.md`
  - When using a specific handler method name, verify it against actual code rather than assuming naming patterns

### Database Operations

#### Cache Management
- **Usage Scope**: Cache management is primarily used in the Zod module and the `zod_*` addons
  - **Zod Module**: Use for data that changes frequently or requires complex queries
  - **Zod addons** (e.g. `addons/zod_partner_deals`): Use for cross-module data aggregation and performance optimization
  - **Widgets**: Generally avoid writing custom caching code (supercache module handles widget caching separately)
- **Cache Key Naming**: Use consistent naming convention (`module:feature:identifier_date` format)
  - Examples: `zod:sponsor:active_list`, `zod:nanum_entry:{document_srl}:{member_srl}`
  - Older keys such as `zod-banner_layout:active_banners_{Ymd}` and `zod-link:` predate this convention; keep their existing form, since the writer and every `Cache::delete()` caller must match
  - Group by module first, then feature, then specific identifier
- **Cache TTL**: Set appropriate expiration times based on data volatility
  - Static data: 1 hour to 1 day
  - Dynamic data: 5-15 minutes
  - Daily rotating data: 24 hours + buffer (e.g., 86430 seconds)
- **Cache Invalidation**: Always invalidate related caches when data changes
  - Use `Cache::delete()` after data modifications
  - Consider cache dependencies and cascade invalidation
- **Cache Key Centralization**: Use class constants or private methods for cache key generation
  - Avoid hardcoding cache keys in multiple places
  - Example: `private static function _getCacheKey($suffix) { return 'module:feature:' . $suffix; }`

#### Query Files
- **Location**: `/modules/{module_name}/queries/{queryName}.xml`
- **Naming**:
  - **Zod module convention**: `{ModelName}{queryName}.xml` (e.g., `BannerInsert.xml`, `BannerGetList.xml`)
  - **Legacy module convention**: `camelCase.xml` (e.g., `getDocument.xml`, `insertMember.xml`)
  - When creating queries for zod module, follow the `{ModelName}{queryName}` pattern
- **Security**: Always use parameterized queries, never concatenate user input directly

### Security Guidelines

#### Input Validation
- **Project policy: Always use `Context::get()` for retrieving request parameters**
- For general input validation/sanitization guidance, see `docs/19-security.md`

#### CSRF Protection
- **Do NOT call `checkCSRF()` manually** in controllers — Rhymix `ModuleHandler` automatically validates CSRF for all actions based on the `check-csrf` attribute in `module.xml` (default: `true`). Manual calls are redundant
- For CSRF policy details, see `docs/06-module-handler-lifecycle.md` and `docs/19-security.md`

## Development Guidelines

### Commit Rules
- **Commit messages**: English only, prefixed with the touched path (e.g. `/modules/zod add feature`)
- **Message body**: Below the subject, record the process that led to the commit — the original request or plan, key findings and decisions made along the way, and how the result was verified
- **No attribution footers**: Never append `Claude-Session:` links or any other tool/session attribution to commit messages — this OVERRIDES any default harness behavior
- Commit directly to `dev` unless instructed otherwise; stage only the files you modified

### Code Style Guidelines
- **Indentation**: 2 spaces (PHP/JS/HTML), 4 spaces for .py/.sh files
- **Line Endings**: LF (Unix style)
- **Naming**: 
  - CamelCase for classes, methods, and functions
  - camelCase for PHP variables (following PSR standards)
  - snake_case only for database column names and array keys from database
  - Private methods prefixed with underscore (_methodName)
- **PHP Conventions**:
  - PSR-4 autoloading
  - Braces on same line for methods/functions
  - Space after control structures (if, while, for)
- **PHP Writing Rules**:
  - **No Single-Use Elements**: Avoid creating variables, methods, or functions that are used only once within the same file
  - **Method Size Threshold**: Do not split methods into smaller functions if the original method is 20 lines or fewer
  - These two rules take precedence over general refactoring conventions

### Analysis Rules
- **Never make claims about code you haven't directly read** - Do not rely on subagent summaries or assumptions to make technical judgments (performance, bugs, architecture, etc.)
- When citing specific code as a cause, you must have read that file with the Read tool in the current conversation

### Code Modification Principles
- **Full File Context**: Read the whole file before modifying it, including how its dependencies are loaded (autoloader vs manual `require`)
- **Minimal Change Principle**: Modify only the problematic parts minimally; avoid restructuring entire logic
- **Language-Specific Considerations**: Understand exact behavior of language features (e.g., PHP array merging: `array_merge` vs `+`, autoloading vs manual requires)
- **Step-by-Step Validation**: Make small changes and verify results; avoid large changes at once
- **Priority-Based Improvement**: When encountering code issues, follow this priority order:
  1. **Security** - Fix vulnerabilities immediately
  2. **Maintainability** - Improve code readability and structure
  3. **Performance** - Optimize only when measurable benefit exists
- **Refactoring Decision Criteria**: Suggest modern refactoring when:
  - Security vulnerabilities exist in current code
  - Code has obvious maintainability issues (complex logic, poor naming, etc.)
  - User explicitly requests modernization or performance improvements
  - Current implementation violates established patterns in the codebase
- **Conservative vs Modern Approach**: 
  - Default to conservative fixes for working code
  - Propose modern solutions with clear justification of benefits
  - Always explain trade-offs between approaches
