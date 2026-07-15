# Contributing

Thank you for your interest in contributing to this project!

This repository documents a working observability lab (Prometheus, Loki, Grafana, cAdvisor, Promtail) and its evolution across three documented phases. Contributions are welcome in the form of bug reports, configuration fixes, documentation improvements, or roadmap feature suggestions.

## How to Contribute

### Reporting Issues

- Use the **Bug Report** template for configuration errors, broken deployments, or documentation that doesn't match the actual repo state.
- Use the **Feature Suggestion** template for ideas related to the [Roadmap](../README.md#roadmap) (alerting, hardening, tracing, host metrics) or entirely new capabilities.

### Submitting Changes

1. Fork the repository.
2. Create a feature branch: `git checkout -b feature/your-feature-name`
3. Before touching `docker-compose.yml` or anything under `config/`, validate your change locally:
   ```bash
   docker compose up -d
   docker compose ps   # all 7 services should report "Up"
   ```
4. Follow the existing documentation style — see any file in [`phases/`](../phases/) for reference (Objective → Implementation → Engineering Log → Final Validation → Verification checklist).
5. Submit a pull request with a clear description of the change and, if applicable, the validation output from step 3.

### Documentation Style

- Each phase document follows a consistent structure: **Objective → Implementation/Setup → Engineering Log (issues + resolutions) → Final Validation → Next Steps**.
- Troubleshooting entries follow the pattern **Issue → Analysis → Resolution**, with a "Lesson learned" callout when the fix generalizes beyond this specific stack.
- Configuration files (`config/**/*.yml`) must stay in sync with what's actually deployed — if you change a mounted config, update the corresponding phase doc's embedded copy too.

### What NOT to open a PR for

- New phases beyond Phase 3 — the core pipeline is intentionally scoped as complete. Larger extensions (alerting, hardening) belong in an issue first so scope can be discussed before implementation.

## Questions?

Feel free to open a Discussion or reach out via [LinkedIn](https://www.linkedin.com/in/alejandro-zavala-zenteno).
