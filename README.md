# Agentic Threat Intel Harness

[![CI](https://github.com/alshaherisam-afk/agentic-threat-intel-harness/actions/workflows/ci.yml/badge.svg)](https://github.com/alshaherisam-afk/agentic-threat-intel-harness/actions/workflows/ci.yml)
[![Python 3.11](https://img.shields.io/badge/python-3.11-blue.svg)](https://www.python.org/downloads/release/python-3110/)
[![Security: Bandit](https://img.shields.io/badge/security-Bandit-yellow.svg)](https://bandit.readthedocs.io/)

A security-focused, asynchronous threat-intelligence harness for validating, sanitizing, and enriching Indicators of Compromise (IOCs).

The system treats IOC context as untrusted input, detects common prompt-injection patterns, validates IPv4/IPv6 and SHA-256 indicators, and routes high-risk results to human approval.

> [!WARNING]
> **Security Notice:** This project is a decision-support component. It must not be used as an autonomous threat verdict or response system without qualified analyst review, authenticated threat feeds, and appropriate operational controls.

## Architecture

```text
[Raw IOC Request]
          |
          v
[Pydantic v2 Validation]
          |
          v
[IOC Sanitizer]
  |                    |
  |                    +--> Invalid IOC -> Rejected
  |
  +--> Prompt injection detected -> Human approval / blocked
          |
          v
[Async Threat Intelligence Enrichment]
          |
          v
[Threat Score Evaluation]
       /          \
      /            \
 Score <= 80     Score > 80
    |                |
    v                v
[Low-Risk       [Human Approval
 Report]          Required]
```

## Security posture

### Input validation

- Pydantic v2 models validate all external request data.
- Unknown fields are rejected.
- Context payloads have a maximum length.
- IPv4, IPv6, and SHA-256 formats are validated before enrichment.

### Prompt-injection defense

- IOC context is treated as untrusted data.
- Instruction-like patterns such as `ignore previous instructions` and fake system messages are detected.
- Detected injection text is replaced with an explicit audit marker.
- Injection findings require human approval and cannot control application behavior.

### Threat scoring

- Threat intelligence feeds return a score from 0 to 100.
- Scores above 80 require human approval.
- Prompt-injection findings are marked as blocked even when the threat score is low.
- Feed responses are treated as data, never as executable instructions.

### Runtime hardening

- The container is based on `python:3.11-slim`.
- The application runs as non-root UID/GID `10001`.
- Kubernetes configuration uses:
  - `readOnlyRootFilesystem: true`
  - `allowPrivilegeEscalation: false`
  - `runAsNonRoot: true`
  - dropped Linux capabilities
  - the RuntimeDefault seccomp profile
- CI runs Bandit, pytest, and Trivy.

## Repository layout

```text
.
├── .github/
│   ├── copilot-instructions.md
│   └── workflows/
│       └── ci.yml
├── src/
│   └── threat_intel/
│       ├── __init__.py
│       ├── enricher.py
│       ├── models.py
│       └── sanitizer.py
├── tests/
│   ├── __init__.py
│   ├── test_enricher.py
│   └── test_sanitizer.py
├── Dockerfile
├── k8s-deployment.yaml
├── pyproject.toml
└── README.md
```

## Local quickstart

Create and activate a virtual environment:

```bash
python -m venv .venv
```

On macOS or Linux:

```bash
source .venv/bin/activate
```

On Windows PowerShell:

```ps1
.venv\Scripts\Activate.ps1
```

Install the package and development dependencies:

```bash
pip install -e ".[dev]"
```

Run the test suite:

```bash
pytest
```

Run Bandit security analysis:

```bash
bandit -r src
```

## Build and run the container

Build the image:

```bash
docker build -t agentic-threat-intel-harness .
```

Run the container:

```bash
docker run --rm agentic-threat-intel-harness
```

## Kubernetes deployment

Update the image reference in `k8s-deployment.yaml` to point to a trusted container registry, then apply the manifest:

```bash
kubectl apply -f k8s-deployment.yaml
```

Check the deployment:

```bash
kubectl get deployments
kubectl get pods
```

## Development workflow

Before committing changes:

```bash
pytest
bandit -r src
git diff --check
```

Stage and commit the README:

```bash
git add README.md
git commit -m "Add threat intelligence harness README"
git push origin main
```

## Production requirements

Before using this project with operational data, add and validate:

- Authentication for threat-intelligence feed requests.
- TLS certificate verification and endpoint allowlisting.
- Request timeouts, retry limits, and rate limiting.
- Signed feed provenance and response integrity checks.
- Secure, tamper-evident audit-log storage.
- Secrets management through environment variables or a dedicated secret manager.
- Analyst authentication and role-based approval workflows.
- Monitoring, alerting, and incident response procedures.
- Legal and security review of all enrichment providers and indicators.
