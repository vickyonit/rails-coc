# The Complete Ruby on Rails Conventions Guide

> **"Convention over Configuration"** — The foundational philosophy that makes Rails productive.

This guide is the definitive, exhaustive reference for every convention a Rails developer must follow. It covers naming, structure, models, controllers, views, routing, databases, testing, security, APIs, background jobs, mailers, frontend, code style, and advanced patterns.

---

## 📚 Table of Contents

| # | File | Topics Covered |
|---|------|---------------|
| 01 | [Naming Conventions](./01-naming-conventions.md) | Classes, files, databases, variables, routes, methods, modules, constants |
| 02 | [Project Structure](./02-project-structure.md) | Directory layout, file organization, autoloading, Zeitwerk, engines |
| 03 | [Models & Active Record](./03-models-activerecord.md) | Associations, validations, callbacks, scopes, enums, STI, polymorphism, queries |
| 04 | [Controllers](./04-controllers.md) | RESTful actions, filters, strong params, concerns, error handling, rendering |
| 05 | [Views & Templates](./05-views-templates.md) | ERB/Haml, partials, layouts, helpers, view components, Turbo/Stimulus |
| 06 | [Routing](./06-routing.md) | RESTful routes, nested resources, namespaces, constraints, concerns |
| 07 | [Database & Migrations](./07-database-migrations.md) | Migrations, schema design, seeds, indexes, foreign keys, data migrations |
| 08 | [Testing Conventions](./08-testing-conventions.md) | Minitest, RSpec, fixtures, factories, system tests, CI practices |
| 09 | [Configuration & Environments](./09-configuration-environments.md) | Credentials, environment configs, initializers, locales, logging |
| 10 | [Security Conventions](./10-security.md) | CSRF, SQL injection, XSS, authentication, authorization, content security |
| 11 | [API Conventions](./11-api-conventions.md) | API mode, versioning, serialization, pagination, rate limiting, documentation |
| 12 | [Background Jobs & Active Job](./12-background-jobs.md) | Job conventions, queues, retries, scheduling, idempotency |
| 13 | [Action Mailer](./13-action-mailer.md) | Mailer naming, views, previews, delivery, interceptors |
| 14 | [Action Cable](./14-action-cable.md) | Channels, connections, subscriptions, broadcasting |
| 15 | [Active Storage & File Handling](./15-active-storage.md) | Attachments, variants, direct uploads, services |
| 16 | [Asset Pipeline & Frontend](./16-assets-frontend.md) | Importmaps, Propshaft, jsbundling, cssbundling, Hotwire |
| 17 | [Code Style & Ruby Conventions](./17-code-style.md) | Ruby idioms, formatting, Rubocop, method design, error handling |
| 18 | [Advanced Patterns](./18-advanced-patterns.md) | Service objects, form objects, query objects, decorators, concerns, POROs |
| 19 | [Performance Conventions](./19-performance.md) | N+1 queries, caching, eager loading, database optimization, profiling |
| 20 | [Deployment & DevOps](./20-deployment-devops.md) | Kamal, Docker, CI/CD, monitoring, logging, environment management |

---

## Rails Version

This guide targets **Rails 7.1+** with notes for **Rails 8.0** where applicable.

## How to Use This Guide

- **New to Rails?** — Start with files 01–07 for core conventions
- **Building an API?** — Focus on files 03, 04, 06, 11
- **Code review checklist?** — Files 17, 19, 10 are your go-to references
- **Scaling up?** — Files 18, 19, 12, 20 cover advanced patterns

## 🤖 AI Agent Skill

This repository includes a **Rails Conventions Skill** for AI coding assistants (Cursor, Codex, etc.) that helps agents follow Rails conventions automatically.

**Installation:**
```bash
cp -r .cursor/skills/rails-conventions ~/.cursor/skills/rails-conventions
```

See [.cursor/skills/rails-conventions/README.md](./.cursor/skills/rails-conventions/README.md) for details.

---

## The Golden Rules of Rails

1. **Convention over Configuration** — Follow the defaults unless you have a strong reason not to
2. **DRY (Don't Repeat Yourself)** — Extract shared logic into concerns, helpers, or service objects
3. **Fat Models, Skinny Controllers** — Business logic belongs in models, not controllers
4. **RESTful Design** — Think in resources, not arbitrary actions
5. **Principle of Least Surprise** — Code should do what a reader expects
6. **Fail Fast, Fail Loud** — Use bang methods and validations to catch errors early
7. **Test Everything That Could Break** — Especially edge cases and business logic
