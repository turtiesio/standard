# Development Rules for NestJS Agent

## Project Configuration

- Package manager: `yarn` (strictly enforced)
- Logger: `pino` (strictly enforced)
- API documentation: Swagger/OpenAPI integration mandatory(NestJS Decorators)
- Documentation generator: Compodoc for comprehensive documentation

## Coding Standards

1. Architecture Principles

- SOLID principles implementation
- Clean Architecture patterns
- Clean Code practices
- KISS (Keep It Simple, Stupid)
- DRY (Don't Repeat Yourself)
- YAGNI (You Aren't Gonna Need It)

1. Code Documentation

- Compodoc integration for all:
  - Classes
  - Methods
  - DTOs
  - Controllers
  - Services
  - Entities
- Inline documentation using JSDoc format
- Clear naming conventions for:
  - Functions (verb + noun: createUser, validateInput)
  - Variables (camelCase, descriptive)
  - Modules (feature-based naming)
  - Classes (PascalCase, descriptive)

1. Testing Framework

- Unit Tests: Jest
- Integration Tests: Jest + Testing Module
- E2E Tests: Supertest
- Test files naming: `*.spec.ts` for unit/integration, `*.e2e-spec.ts` for E2E

1. Development Tools

- TypeScript (strict mode enabled)
- ESLint + Prettier configuration
- Husky for git hooks
- Logging: JSON format using Winston/Pino
- Makefile for common commands:

```bash
build:
format:
start:
start-dev:
start-debug:
start-prod:
lint:
test:
test-watch:
test-cov:
test-debug:
test-e2e:
```

1. write and update readme README

1. Logging Standards

```typescript
{
  timestamp: ISO8601,
  level: 'log' | 'fatal' | 'error' | 'warn' | 'debug' | 'verbose',
  context: string,
  message: string,
  metadata: Record<string, any>
}

// fatal: 'Critical error requiring application shutdown',
// error: 'Business logic failures, exceptions thrown',
// warn: 'Potential issues, performance degradation',
// info: 'Important business events',
// debug: 'Detailed information for development debugging',
// verbose: 'Most detailed application operational information'
```

- JSON structured logging
- Detailed metadata logging
- Redact sensitive information
- Use debug and verbose levels where appropriate
