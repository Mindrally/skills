---
name: ruby-rails
description: "Conventions and best practices for building web applications with Ruby on Rails. Use when scaffolding Rails apps or generators, designing ActiveRecord models and migrations, wiring up Hotwire/Turbo/Stimulus interactivity, setting up background jobs or caching, or writing RSpec/Minitest coverage for Rails code."
---

# Ruby on Rails

This skill covers Ruby and Ruby on Rails development, including Rails conventions, ActiveRecord patterns, frontend integration, security, and testing.

## Core Principles

- Write concise, idiomatic Ruby code with accurate examples
- Adhere to Rails conventions (Convention over Configuration)
- Follow the Ruby Style Guide for formatting consistency
- Leverage Ruby 3.x features like pattern matching and endless methods
- Prefer Rails command-line generators (`bin/rails generate ...`) over hand-written boilerplate — they keep files consistent with framework conventions and wire up tests/routes automatically
- Use `bin/dev` to run the local development server (it boots the web process plus any configured watchers) and check logs after significant changes

## Naming Conventions

- Use snake_case for files, methods, and variables
- Use CamelCase for classes and modules
- Follow Rails naming conventions for models, controllers, views

## Architecture & Performance

- Utilize ActiveRecord for database operations with proper indexing
- Use PostgreSQL in production, with proper indexes, connection pooling, and safe/reversible migrations
- Implement eager loading to prevent N+1 query problems
- Apply fragment caching and Russian Doll caching strategies
- Use service objects for complex business logic
- Follow MVC architecture strictly
- Keep controllers RESTful and focused

## Rails 8 Defaults

Rails 8 ships with a "no PaaS needed" stack — favor these built-ins over adding external services unless there's a specific reason not to:

- **Solid Queue** — database-backed ActiveJob adapter; use it as the default background job backend instead of standing up Redis/Sidekiq unless job volume or feature needs (e.g., Sidekiq Pro batches) require it
- **Solid Cache** — database-backed cache store; a reasonable default for Rails.cache without a separate Redis/Memcached deployment
- **Solid Cable** — database-backed Action Cable adapter for WebSockets without Redis
- **Propshaft** — the default asset pipeline; simpler than Sprockets, just serves fingerprinted files without a bundling step
- **Kamal** — the default deployment tool for shipping containerized Rails apps to your own servers

## Frontend & UI

- Employ Hotwire (Turbo and Stimulus) for standard Rails interactivity and dynamic interactions without full page reloads
- Reach for a Vite-managed JS pipeline (e.g., vite_rails) only when you actually need npm-managed JavaScript (React/Vue components, complex bundling) — don't add it by default
- Design responsively with Tailwind CSS
- Maintain DRY views through helpers and partials
- Use ViewComponents for reusable UI components

## Security

- Implement authentication/authorization via Devise or Pundit
- Use strong parameters in controllers to prevent mass assignment vulnerabilities
- Sanitize user inputs appropriately
- Use CSRF protection tokens
- Implement proper session management

## Testing

- Write comprehensive Minitest or RSpec coverage for models, controllers, and integration flows, following TDD practices
- Use FactoryBot for test data generation rather than fixtures
- Mock external services; stub predefined return values
- Use shared examples for common behaviors across different contexts
- Ensure each test is independent; avoid shared state between tests

## Best Practices

- Keep controllers thin, models fat (but not too fat)
- Use concerns for shared functionality
- Implement background jobs with Sidekiq or ActiveJob
- Use proper database migrations
- Follow RESTful routing conventions
