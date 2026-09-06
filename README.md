# CI/CD Pipeline Security – Research Demonstration

This repository accompanies a research paper on CI/CD pipeline security. It
contains a minimal Spring Boot application and two GitHub Actions workflows
that illustrate the contrast between common insecure practices and a hardened
alternative.

> **Warning:** `.github/workflows/insecure.yml` is an intentionally vulnerable
> workflow. It exists for educational and research purposes only. It must never
> be copied into a production repository.

---

## Repository structure

```
.
├── pom.xml                                      # Spring Boot 3 / Java 17 Maven project
├── src/
│   ├── main/java/com/example/demo/
│   │   ├── DemoApplication.java                 # Spring Boot entry point
│   │   └── HelloController.java                 # GET /hello → "Hello World"
│   └── test/java/com/example/demo/
│       └── HelloControllerTest.java             # MockMvc test for the controller
├── demo/
│   └── fake-secrets-for-trufflehog-demo.txt    # Non-functional dummy credential for scanning demo
└── .github/workflows/
    ├── insecure.yml                             # Anti-example: intentionally vulnerable
    └── secure.yml                               # Hardened reference workflow
```

---

## Spring Boot application

The application is intentionally minimal. It exists only so the workflows have
a real Maven project to compile, package, and test. The single REST endpoint
`GET /hello` returns the string `Hello World`. The `@WebMvcTest` test verifies
the response status and body without starting a full application context.

Build and run locally:

```bash
mvn -B verify          # compile, test, package
mvn spring-boot:run    # start on http://localhost:8080
curl http://localhost:8080/hello
```

---

## Workflows

### `insecure.yml` – Anti-example (intentionally vulnerable)

| Flaw | Location in file | Risk |
|---|---|---|
| **Poisoned Pipeline Execution (PPE)** | `on: pull_request_target` + `ref: head.sha` checkout | Untrusted PR code runs with access to repository secrets |
| **Mutable action references** | `actions/checkout@v4`, `actions/setup-java@v4` | A tag can be silently redirected to malicious code |
| **No permissions block** | Absence of `permissions:` key | `GITHUB_TOKEN` inherits broad default write permissions |
| **Script injection** | `echo "Building PR: ${{ github.event.pull_request.title }}"` | Attacker-controlled input is expanded by the shell before sanitisation |

Each flaw is annotated in the file with a detailed English comment explaining
the mechanism and the conditions under which it can be exploited.

### `secure.yml` – Hardened reference

| Control | What it does | Risk mitigated |
|---|---|---|
| **`pull_request` trigger** | Runs in a restricted context with no secret access | Poisoned Pipeline Execution |
| **Pinned action SHAs** | Each action is locked to a 40-character commit hash | Supply-chain compromise via tag hijacking |
| **`permissions: contents: read`** | Deny-by-default token scope at workflow level | Overly permissive `GITHUB_TOKEN` |
| **Temurin JDK + Maven cache** | Reproducible toolchain with reduced external network calls | Dependency confusion, build inconsistency |
| **Environment variable indirection** | Attacker-controlled values assigned to `env:` before use in `run:` | Script injection |
| **zizmor** | Static analysis of workflow files on every PR | Misconfigured workflow patterns |
| **TruffleHog** | Scans commits for credential patterns | Accidental or intentional secret exposure |

---

## Demo: TruffleHog detection

The file `demo/fake-secrets-for-trufflehog-demo.txt` contains
`AKIAIOSFODNN7EXAMPLE`, which is AWS's own publicly documented example key
(non-functional, published in AWS documentation). It is included so the
TruffleHog step in `secure.yml` has a detectable pattern to report. In a real
repository this file would be removed or added to TruffleHog's allowlist as a
known false positive.

---

## Security controls summary

The table below maps each vulnerability class to the workflow that demonstrates
it and the corresponding defensive control.

| Vulnerability | Insecure workflow | Secure workflow |
|---|---|---|
| Poisoned Pipeline Execution | `pull_request_target` + untrusted checkout | `pull_request` with no secret access |
| Supply-chain via mutable tags | `@v4` tags | Full commit SHA pins |
| Excessive token permissions | No `permissions` block | `permissions: contents: read` |
| Script injection | Direct `${{ expression }}` in `run:` | Intermediate `env:` variable |
| Unaudited workflow changes | No static analysis | zizmor on every PR |
| Credential leakage | No scanning | TruffleHog on every PR |
