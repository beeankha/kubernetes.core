# SonarCloud

Dashboard:

[SonarCloud project overview](https://sonarcloud.io/project/overview?id=ansible-collections_kubernetes.core)

## CI integration

CI runs the **`all_green`** workflow (linters on pull requests, sanity, units, then a **coverage** job
that uploads **`coverage*`** artifacts), then **`SonarCloud`** runs on **`workflow_run`** and passes
**`sonar.python.coverage.reportPaths`** to the scanner. That matches
[ansible-collections/amazon.aws#2871](https://github.com/ansible-collections/amazon.aws/pull/2871).

Workflow files:

- [.github/workflows/all_green_check.yaml](.github/workflows/all_green_check.yaml)
- [.github/workflows/sonarcloud.yml](.github/workflows/sonarcloud.yml)

Scanner configuration lives in [sonar-project.properties](sonar-project.properties).

The **coverage** job uses **`ansible-test`** (`units --coverage`, then **`coverage combine`** /
**`coverage xml`**), then writes **`coverage.xml`** with workspace paths normalized for Sonar.
**`pytest-cov`** is listed in **`tests/unit/requirements.txt`** for parity and any direct pytest runs;
**`ansible-test`** still owns the coverage data used in CI.

Org secrets and fork PR behavior follow GitHub's
[secrets in Actions](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions)
documentation; the Sonar finalize job runs in this repository after **`all_green`** succeeds.

The **`finalize`** job only runs when **`workflow_run.head_repository`** matches **`ansible-collections/kubernetes.core`**, so we do not checkout arbitrary **`head_sha`** commits from forks while using org **`SONAR_TOKEN`**. Pull requests opened from forks still run **`all_green`** for CI; they do not trigger this Sonar finalize job (same limitation as typical fork secret behavior).
