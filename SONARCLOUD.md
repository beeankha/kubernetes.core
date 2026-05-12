# SonarCloud

Dashboard:

[SonarCloud project overview](https://sonarcloud.io/project/overview?id=ansible-collections_kubernetes.core)

## CI integration

Sonar analysis is implemented in **[.github/workflows/sonarcloud.yml](.github/workflows/sonarcloud.yml)** as a **reusable workflow** (`on: workflow_call` only). It is **not** triggered by `workflow_run`.

The caller (for example **[.github/workflows/all_green_check.yaml](.github/workflows/all_green_check.yaml)**) should:

1. Run **linters** (on pull requests), **sanity**, **units**, and **coverage**, then pass an aggregate **`all_green`** gate.
2. Upload a **`coverage`** artifact from the **coverage** job (single XML at the repo root after path normalization).
3. Invoke **`uses: ./.github/workflows/sonarcloud.yml`** with **`secrets: inherit`** (or pass **`ANSIBLE_COLLECTIONS_ORG_SONAR_TOKEN_CICD_BOT`** explicitly) **after** **`all_green`** and **`coverage`** succeed, in the **same** workflow run so **`actions/download-artifact`** can fetch **`coverage`**.

Because the caller runs on **`pull_request`** or **`push`**, the reusable workflow inherits that **`github.event`**. **`actions/checkout`** uses **`github.event.pull_request.head.sha`** on pull requests and **`github.sha`** on push, which matches Sonar's guidance for avoiding **`workflow_run`** + **`workflow_run.head_sha`** checkout patterns. PR parameters (**`sonar.pullrequest.*`**) are taken from **`github.event.pull_request`** (no `gh` API calls in **`sonarcloud.yml`**).

The scan step uses **`SonarSource/sonarqube-scan-action`** (pinned SHA in the workflow file) with **`sonar.python.coverage.reportPaths`** set from any **`coverage*.xml`** files found under the workspace after the artifact download. The overall flow (coverage in CI, then Sonar with XML) follows the same idea as [ansible-collections/amazon.aws#2871](https://github.com/ansible-collections/amazon.aws/pull/2871). Wire **`sonarcloud.yml`** from **`all_green`** with **`workflow_call`** (instead of a separate **`workflow_run`** finalize workflow) once the **`sonarcloud`** job is added to **`all_green_check.yaml`**.

Workflow files:

- [.github/workflows/all_green_check.yaml](.github/workflows/all_green_check.yaml) — expected caller: **`all_green`** gate, **coverage** artifact upload, and a **`sonarcloud`** job with **`uses: ./.github/workflows/sonarcloud.yml`** and **`secrets: inherit`** (after **`all_green`** and **`coverage`**), gated for **`push`** and same-repo **`pull_request`** when the org Sonar secret is set.
- [.github/workflows/sonarcloud.yml](.github/workflows/sonarcloud.yml) — **`scan`** job: checkout, download **`coverage`**, **`SONAR_ARGS`**, SonarCloud scan.

Scanner configuration lives in [sonar-project.properties](sonar-project.properties).

The **coverage** job (in **`all_green`**) uses **`ansible-test`** (`units --coverage`, then **`coverage combine`** / **`coverage xml`**), then writes **`coverage.xml`** with workspace paths normalized for Sonar. **`pytest-cov`** is listed in **`tests/unit/requirements.txt`** for parity and any direct pytest runs; **`ansible-test`** still owns the coverage data used in CI.

**`sonarcloud.yml`** declares a required secret **`ANSIBLE_COLLECTIONS_ORG_SONAR_TOKEN_CICD_BOT`** and **`permissions: contents: read`**, **`pull-requests: read`**.

Org secrets and fork PR behavior follow GitHub's [secrets in Actions](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions) documentation. The **caller** should **`if:`**-gate the reusable Sonar job (for example: same-repo pull requests and/or **`push`**, and secret present) so org tokens are not used for fork-head checkouts; fork PRs can still run **`all_green`** for CI without running Sonar.
