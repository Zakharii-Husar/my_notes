# Next.js Top-Level Files

Top-level files in Next.js are used to configure your application, manage dependencies, run proxy servers, integrate monitoring tools, and define environment variables. These files are typically placed in the root directory of your Next.js project.

## Configuration Files

| File | Purpose |
|------|---------|
| `next.config.js` | Main configuration file for Next.js - customizes build process, routing, and runtime behavior |
| `package.json` | Defines project dependencies, scripts, and metadata for npm/yarn |
| `instrumentation.ts` | Sets up OpenTelemetry and instrumentation for monitoring and observability |
| `proxy.ts` | Configures Next.js request proxy for API routing and CORS handling |
| `eslint.config.mjs` | Defines ESLint rules and configuration for code linting and style enforcement |
| `tsconfig.json` | TypeScript compiler configuration and project settings |
| `jsconfig.json` | JavaScript project configuration (alternative to tsconfig.json for JS projects) |

## Environment Variables

| File | Purpose |
|------|---------|
| `.env` | Global environment variables accessible in all environments |
| `.env.local` | Local development environment variables (highest priority, not committed to git) |
| `.env.production` | Environment variables specific to production deployment |
| `.env.development` | Environment variables specific to development environment |

## Git and TypeScript

| File | Purpose |
|------|---------|
| `.gitignore` | Specifies files and directories that Git should ignore |
| `next-env.d.ts` | TypeScript declarations for Next.js types (auto-generated, not tracked by git) |
