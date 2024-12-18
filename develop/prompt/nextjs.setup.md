# NextJS Setup

초기 셋업

Lets implement

1. prettier(use default config),
2. husky(git hooks. use 'echo {script} >> .husky/{HOOK_NAME}'. no husky install or husky add),
3. lint-staged,
4. commit lint(release automation with sementaic versionign)

```bash
yarn add -D prettier husky lint-staged @commitlint/cli @commitlint/config-conventional comment-lint
```
