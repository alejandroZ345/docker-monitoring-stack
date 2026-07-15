## Description

<!-- What does this PR change, and why? -->

## Type of change

- [ ] Bug fix (configuration or documentation correction)
- [ ] New feature (roadmap item or new capability)
- [ ] Documentation only
- [ ] Other (describe below)

## Validation

<!-- If this touches docker-compose.yml or config/, confirm the stack still deploys cleanly -->

```bash
docker compose up -d
docker compose ps
```

- [ ] All 7 containers report `Up` after this change
- [ ] No new secrets or credentials were committed
- [ ] If a config file changed, the corresponding phase doc's embedded copy was updated to match

## Additional context

<!-- Screenshots, related issues, anything else a reviewer should know -->
