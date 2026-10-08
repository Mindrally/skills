# Claude Code Skills Collection

**265+ Claude Code skills converted from Cursor rules. Supercharge your AI coding with expert guidelines for React, Python, TypeScript, and everything in between.**

[![GitHub stars](https://img.shields.io/github/stars/Mindrally/skills?style=flat&logo=github)](https://github.com/Mindrally/skills/stargazers)
[![Skills](https://img.shields.io/badge/skills-265-blue)](#available-skills)
[![skills.sh](https://skills.sh/b/Mindrally/skills)](https://skills.sh/Mindrally/skills)
[![License](https://img.shields.io/github/license/Mindrally/skills)](LICENSE)
[![Last commit](https://img.shields.io/github/last-commit/Mindrally/skills)](https://github.com/Mindrally/skills/commits/main)

```bash
npx skills add Mindrally/skills --skill react
```

A comprehensive collection of skills for [Claude Code](https://claude.ai/code), Anthropic's official CLI for Claude. Maintained by [Mindrally](https://mindrally.com).

## Origin

These skills were converted from **Cursor Rules** (`.cursorrules` files) to the Claude Code skills format (`SKILL.md`). The original Cursor rules provided coding guidelines and best practices that Cursor AI would follow when generating code.

Each skill has been reformatted to work with Claude Code's skill system, which uses YAML frontmatter for metadata and markdown for the actual guidelines.

## What Are Skills?

Skills are reusable instruction sets that enhance Claude Code's capabilities for specific technologies, frameworks, or development practices. When activated, they provide Claude with expert-level context about:

- Coding conventions and best practices
- Framework-specific patterns
- Project structure guidelines
- Error handling approaches
- Performance optimization techniques

## Skill Format

Each skill follows this structure:

```markdown
---
name: skill-name
description: Brief description of the skill
---

# Skill Title

Guidelines and instructions...
```

## Installation

### Using `npx skills`

The [`skills` CLI](https://github.com/vercel-labs/skills) installs skills from this repo directly into Claude Code (and 70+ other agents) with no setup. It requires Node.js.

Install a single skill into the current project:

```bash
npx skills add Mindrally/skills --skill react
```

Install several skills at once:

```bash
npx skills add Mindrally/skills --skill react --skill typescript --skill tailwindcss
```

Install globally for all your projects (`~/.claude/skills/`), targeting Claude Code only, with no prompts:

```bash
npx skills add Mindrally/skills --skill react -g -a claude-code -y
```

Browse what's available before installing:

```bash
npx skills add Mindrally/skills --list
```

Install everything:

```bash
npx skills add Mindrally/skills --skill '*' -a claude-code
```

Other useful commands:

```bash
npx skills ls                  # list installed skills
npx skills update              # update installed skills to the latest version
npx skills remove react        # remove a skill
```

### Skill bundles

One command for a whole stack:

```bash
# Next.js full-stack
npx skills add Mindrally/skills --skill nextjs-react-typescript --skill tailwindcss --skill prisma --skill zod-schema-validation --skill jest

# Python backend
npx skills add Mindrally/skills --skill python --skill fastapi-python --skill postgresql-best-practices --skill python-testing --skill docker

# Python data science
npx skills add Mindrally/skills --skill pandas-best-practices --skill numpy-best-practices --skill scikit-learn-best-practices --skill matplotlib-best-practices --skill data-analysis-jupyter

# Go microservices
npx skills add Mindrally/skills --skill go --skill go-backend-microservices --skill grpc-development --skill kubernetes --skill docker

# React Native
npx skills add Mindrally/skills --skill expo-react-native-typescript --skill react-query --skill zustand-state-management --skill jest

# Infrastructure
npx skills add Mindrally/skills --skill terraform --skill kubernetes --skill docker --skill ci-cd-best-practices --skill monitoring-guidelines

# Quality baseline for any project
npx skills add Mindrally/skills --skill clean-code --skill testing --skill security-best-practices --skill git-workflow --skill readme-best-practices
```

### Claude Code plugin marketplace

The repo is also a Claude Code plugin marketplace. Inside a Claude Code session:

```
/plugin marketplace add Mindrally/skills
/plugin install mindrally-skills@mindrally-skills
```

This installs the whole library as one plugin and keeps it updated through the plugin system.

## Available Skills

### Frontend Frameworks
| Skill | Description |
|-------|-------------|
| `react` | React development with hooks, performance optimization |
| `nextjs-react-typescript` | Next.js with React and TypeScript |
| `vue-typescript` | Vue.js with TypeScript |
| `angular` | Angular framework development |
| `svelte` | Svelte framework |
| `sveltekit` | SvelteKit full-stack framework |
| `remix` | Remix framework |
| `astro` | Astro static site generator |
| `nuxtjs-vue-typescript` | Nuxt.js with Vue and TypeScript |
| `tanstack-start` | TanStack Start server functions, SSR/streaming, and API routes |
| `tanstack-router` | Type-safe file-based routing with TanStack Router |
| `react-router-v7` | React Router v7 framework-mode route modules, loaders, and actions |

### Mobile Development
| Skill | Description |
|-------|-------------|
| `react-native-cursor-rules` | React Native development |
| `expo-react-native-typescript` | Expo with React Native and TypeScript |
| `flutter` | Flutter cross-platform development |
| `swift` | Swift language development |
| `swiftui-development` | SwiftUI interface development |
| `android-development` | Android native development |
| `kotlin-development` | Kotlin development |
| `ionic` | Ionic hybrid mobile apps |
| `harmony-arkts` | HarmonyOS development with ArkTS and ArkUI |

### Backend & APIs
| Skill | Description |
|-------|-------------|
| `nodejs-development` | Node.js backend development |
| `express-typescript` | Express.js with TypeScript |
| `fastapi-python` | FastAPI Python framework |
| `django-python` | Django web framework |
| `flask-python` | Flask Python framework |
| `ruby-rails` | Ruby on Rails |
| `laravel` | Laravel PHP framework |
| `spring-boot` | Spring Boot Java framework |
| `go-backend-microservices` | Go microservices |
| `nestjs-clean-typescript` | NestJS with clean architecture |
| `graphql` | GraphQL API development |
| `grpc-development` | gRPC development |
| `trpc` | tRPC type-safe APIs |

### Languages
| Skill | Description |
|-------|-------------|
| `typescript` | TypeScript best practices |
| `python` | Python development |
| `go` | Go language |
| `rust` | Rust programming |
| `java` | Java development |
| `c-sharp` | C# development |
| `ruby` | Ruby development |
| `php-development` | PHP development |
| `elixir` | Elixir functional programming |
| `julia` | Julia scientific computing |
| `lua` | Lua scripting |
| `cpp` | C++ development |
| `fortran` | Modern Fortran scientific and numerical computing |

### Databases & ORMs
| Skill | Description |
|-------|-------------|
| `prisma` | Prisma ORM |
| `drizzle-orm` | Drizzle ORM |
| `sequelize` | Sequelize ORM |
| `typeorm` | TypeORM |
| `mongodb-development` | MongoDB |
| `postgresql-best-practices` | PostgreSQL |
| `mysql-best-practices` | MySQL |
| `redis-best-practices` | Redis |
| `elasticsearch-best-practices` | Elasticsearch |
| `supabase` | Supabase backend |

### Data Engineering
| Skill | Description |
|-------|-------------|
| `pyspark-etl` | Performant, testable PySpark ETL pipelines |
| `snowflake-data-engineering` | Snowflake SQL, Dynamic Tables, Streams, Tasks, and Snowpipe |
| `snowflake-cortex-ai` | Snowflake Cortex AI Functions and Cortex Search for in-warehouse RAG |
| `snowflake-snowpark-dbt` | Snowpark Python and dbt with the dbt-snowflake adapter |

### DevOps & Infrastructure
| Skill | Description |
|-------|-------------|
| `docker` | Docker containerization |
| `kubernetes` | Kubernetes orchestration |
| `terraform` | Terraform IaC |
| `aws-development` | AWS cloud development |
| `gcp-development` | Google Cloud Platform |
| `azure` | Microsoft Azure |
| `ci-cd-best-practices` | CI/CD pipelines |
| `github-workflow` | GitHub Actions workflows |
| `gitlab-workflow` | GitLab CI/CD |
| `serverless` | Serverless architecture |

### Testing
| Skill | Description |
|-------|-------------|
| `testing` | General testing practices |
| `jest` | Jest testing framework |
| `cypress` | Cypress E2E testing |
| `playwright` | Playwright testing |
| `python-testing` | Python testing |
| `rspec` | RSpec for Ruby |

### AI & Machine Learning
| Skill | Description |
|-------|-------------|
| `deep-learning` | Deep learning practices |
| `pytorch` | PyTorch framework |
| `langchain-development` | LangChain LLM development |
| `llamaindex-development` | LlamaIndex development |
| `openai-api-development` | OpenAI API integration |
| `anthropic-claude-development` | Claude API development |
| `transformers-huggingface` | Hugging Face Transformers |
| `machine-learning` | ML best practices |
| `computer-vision-opencv` | OpenCV computer vision |
| `nlp-natural-language-processing` | NLP development |
| `tensorflow-deep-learning` | TensorFlow and Keras model building, training, and deployment |
| `automl-hyperparameter-optimization` | AutoML and hyperparameter search with Optuna, Ray Tune, and PyCaret |
| `google-adk` | Building AI agents with Google's Agent Development Kit |

### Styling & UI
| Skill | Description |
|-------|-------------|
| `tailwindcss` | Tailwind CSS |
| `css` | CSS best practices |
| `sass-best-practices` | Sass/SCSS |
| `styled-components-best-practices` | Styled Components |
| `framer-motion` | Framer Motion animations |
| `three-js` | Three.js 3D graphics |
| `design-systems` | Design system development |
| `ui-design` | UI design principles |
| `ux-design` | UX design principles |

### Build Tools
| Skill | Description |
|-------|-------------|
| `vite` | Vite build tool |
| `webpack-bundler` | Webpack |
| `esbuild-bundler` | esbuild |
| `parcel-bundler` | Parcel |
| `rollup-bundler` | Rollup |
| `turbopack-bundler` | Turbopack |

### Authentication
| Skill | Description |
|-------|-------------|
| `auth0-authentication` | Auth0 |
| `nextauth-authentication` | NextAuth.js |
| `clerk-authentication` | Clerk |
| `oauth-implementation` | OAuth |
| `jwt-security` | JWT security |

### Blockchain & Web3
| Skill | Description |
|-------|-------------|
| `ethereum` | Ethereum development |
| `solidity` | Solidity smart contracts |
| `solana` | Solana development |
| `blockchain` | General blockchain |
| `onchainkit` | OnchainKit |

### State Management
| Skill | Description |
|-------|-------------|
| `zustand-state-management` | Zustand |
| `redux-toolkit` | Redux Toolkit |
| `react-query` | React Query |
| `tanstack-query` | TanStack Query |
| `swr` | SWR data fetching |
| `vue-pinia` | Vue 3 state management with Pinia setup stores |

### CMS & E-commerce
| Skill | Description |
|-------|-------------|
| `wordpress` | WordPress development |
| `shopify` | Shopify development |
| `woocommerce` | WooCommerce |
| `drupal-development` | Drupal |
| `sanity` | Sanity CMS |
| `ghost` | Ghost CMS |
| `medusa-development` | Medusa v2 commerce modules, workflows, and API routes |

### Specialized Platforms
| Skill | Description |
|-------|-------------|
| `embedded-stm32` | Embedded C/C++ on STM32 microcontrollers with the HAL |
| `ros2-robotics` | ROS 2 robotics nodes, topics, services, and actions |
| `blender-python-addon` | Blender add-on development with the bpy API |
| `gamemaker-gml` | GameMaker Language (GML) game development |

### Utilities & Best Practices
| Skill | Description |
|-------|-------------|
| `git-workflow` | Git best practices |
| `gitflow` | Gitflow branching, versioning, and release workflow |
| `pr-review` | Focused, severity-ranked pull request review |
| `clean-code` | Clean-code principles with anti-over-engineering discipline |
| `readme-best-practices` | Structure and tone guidance for effective READMEs |
| `network-troubleshooting` | Safety-first, read-only network failure diagnosis |
| `security-devsecops` | Secure SDLC, AppSec, and DevSecOps pipeline practices |
| `rtl-internationalization` | Right-to-left layout and bidirectional text support |
| `security-best-practices` | Security guidelines |
| `performance-optimization` | Performance tuning |
| `accessibility-a11y` | Accessibility |
| `seo-best-practices` | SEO optimization |
| `technical-writing` | Technical documentation |
| `logging-best-practices` | Logging practices |
| `observability-guidelines` | Observability |
| `internationalization-i18n` | i18n |
| `localization-l10n` | l10n |

### Complete List (265 Skills)

<details>
<summary>Click to expand full list</summary>

- accessibility-a11y
- alpine-js
- analytics-data-analysis
- android-development
- angular
- angular-development
- anime-js
- anthropic-claude-development
- api-development
- apollo-graphql
- aspnet-core
- astro
- auth0-authentication
- autogen-development
- automl-hyperparameter-optimization
- aws-development
- azure
- backend-development
- bash-scripting
- beautifulsoup-parsing
- bitbucket-workflow
- blazor
- blender-python-addon
- blockchain
- bootstrap
- business-central-development
- c-sharp
- cheerio-parsing
- chrome-extension-development
- ci-cd-best-practices
- clean-architecture
- clean-code
- clerk-authentication
- cloudflare-development
- computer-vision-opencv
- convex
- cpp
- css
- cypress
- data-analysis-jupyter
- data-analyst
- data-jupyter-python
- deep-learning
- deep-learning-python
- deep-learning-pytorch
- deno-typescript
- design-systems
- devops
- django-python
- django-rest-api-development
- docker
- dotnet
- drizzle-orm
- drupal-development
- elasticsearch-best-practices
- electron-development
- elixir
- embedded-stm32
- esbuild-bundler
- ethereum
- expo-react-native-javascript-best-practices
- expo-react-native-typescript
- express-typescript
- fastapi-microservices-serverless
- fastapi-python
- fastify-typescript
- figma-integration
- firebase-development
- flask-python
- flutter
- fortran
- fpga
- framer-motion
- front-end-developer
- game-development
- gamemaker-gml
- gcp-development
- general-best-practices
- ghost
- git-workflow
- gitflow
- github-workflow
- gitlab-workflow
- go
- go-api-development
- go-backend-microservices
- google-adk
- graalvm
- graphql
- graphql-development
- grpc-development
- gsap
- harmony-arkts
- hono-typescript
- html
- htmx
- internationalization-i18n
- ionic
- java
- java-quarkus-development
- java-spring-development
- jax-best-practices
- jest
- julia
- jwt-security
- kafka-development
- koa-typescript
- kotlin-development
- kubernetes
- kysely
- langchain-development
- laravel
- laravel-development
- lerna
- less-best-practices
- llamaindex-development
- llm
- localization-l10n
- logging-best-practices
- lottie
- lua
- machine-learning
- matplotlib-best-practices
- medusa-development
- meta-prompt
- micronaut
- microservices
- modern-web-development
- mongodb-development
- monitoring-guidelines
- monorepo
- monorepo-tamagui
- motion
- mqtt-development
- mysql-best-practices
- nestjs-clean-typescript
- netlify-development
- network-troubleshooting
- nextauth-authentication
- nextjs-react-redux-typescript-cursor-rules
- nextjs-react-typescript
- nextjs-typescript-tailwindcss-supabase
- nlp-natural-language-processing
- nodejs-development
- numpy-best-practices
- nuxtjs-vue-typescript
- nx
- oauth-implementation
- observability-guidelines
- odoo-development
- onchainkit
- openai-api-development
- optimized-nextjs-typescript
- pandas-best-practices
- parcel-bundler
- performance-optimization
- phoenix
- php-development
- pixi-js
- playwright
- playwright-cursor-rules
- pnpm
- postcss-best-practices
- postgresql-best-practices
- pr-review
- prisma
- prisma-development
- puppeteer-automation
- pwa-development
- pyspark-etl
- python
- python-cybersecurity-tool-development
- python-odoo-cursor-rules
- python-testing
- python-uv
- pytorch
- quarkus
- rabbitmq-development
- react
- react-native-cursor-rules
- react-native-r3f
- react-query
- react-router-v7
- readme-best-practices
- redis-best-practices
- redux-toolkit
- remix
- responsive-design
- rest-api-django
- robocorp-cursor-rules
- rollup-bundler
- ros2-robotics
- rspec
- rtl-internationalization
- ruby
- ruby-rails
- rust
- salesforce-development
- salesforce-dx
- sanity
- sass-best-practices
- scikit-learn-best-practices
- scipy-best-practices
- scrapy-web-scraping
- scss-best-practices
- security-best-practices
- security-devsecops
- selenium-automation
- seo-best-practices
- sequelize
- serverless
- shopify
- shopify-theme-development-guidelines
- snowflake-cortex-ai
- snowflake-data-engineering
- snowflake-snowpark-dbt
- solana
- solidity
- spring-boot
- spring-framework
- sql-best-practices
- storybook
- stripe
- styled-components-best-practices
- supabase
- supabase-development
- svelte
- sveltekit
- swift
- swiftui-development
- swr
- systemverilog
- tailwindcss
- tanstack-query
- tanstack-router
- tanstack-start
- tauri-development
- technical-writing
- tensorflow-deep-learning
- terraform
- testing
- three-js
- transformers-huggingface
- trpc
- turbopack-bundler
- turborepo
- typeorm
- typescript
- ui-design
- unity
- ux-design
- vercel-development
- viewcomfy-api-rules
- vite
- vue-pinia
- vue-typescript
- vuejs-typescript-best-practices
- web-development
- web-scraping
- webpack-bundler
- websocket-development
- woocommerce
- wordpress
- zod-schema-validation
- zustand-state-management

</details>

## Usage

Once installed, skills are automatically available in Claude Code. You can reference them in your conversations or they may be automatically applied based on your project context.

## Contributing

Contributions are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) for how to add or improve a skill, or open a [skill request](https://github.com/Mindrally/skills/issues/new?template=skill_request.yml). Changes are tracked in the [CHANGELOG](CHANGELOG.md).

## License

Apache License 2.0. See [LICENSE](LICENSE). Use, modify, and redistribute freely with attribution.

## Credits

- Original Cursor rules from the open-source community, including [PatrickJS/awesome-cursorrules](https://github.com/PatrickJS/awesome-cursorrules)
- Converted and curated for Claude Code compatibility

---

Maintained by [Mindrally](https://mindrally.com), a digital product design and build studio in Austin, Texas. If this library saves you time, a star helps other developers find it.
