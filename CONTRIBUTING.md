# Contributing

## Git workflow

- **Never commit directly to `main`.** Create a short-lived branch, open a pull request, and merge it with squash once checks pass.
- **Branch names:** `type/short-description`, for example `docs/pr-template` or `feat/laravel-install`. Types: `feat`, `fix`, `docs`, `build`, `test`, `ci`, `refactor`, `chore`.
- **Commit messages:** `type(scope): description`, imperative mood, lowercase, no trailing period, for example `build(docker): add php-fpm image`. The scope is optional.
- **Pull request titles** follow the same format, since the squash commit on `main` takes the title of the pull request.
- A change is done only when it comes with its tests. No secrets in Git: `.env` is ignored, `.env.example` is versioned.
