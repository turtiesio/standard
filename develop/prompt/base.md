# Prompting 요구사항

1. Package manager: `yarn`(strictly enforced)
2. Logger: `pino` (strictly enforced)
3. Config: `.env` (strictly enforced)
4. API documentation: Swagger/OpenAPI integration mandatory(NestJS Decorators)
5. Documentation generator: Compodoc for comprehensive documentation
6. Husky for git hooks.
   - No `husky install` or `husky add`.
   - use `echo <commands> >> .husky/<hook_name>`
