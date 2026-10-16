# cybersecurity-architecture-lab

Public MkDocs site at `/cybersecurity-architecture-lab/`. Group rules in `../CLAUDE.md` apply.

## Purpose
Show security-architecture capability bridging infrastructure and security: designs, threat models and control mappings — the evidence base for a Security Architect / CISO track.

## Content model
- Threat models: STRIDE or attack-tree, with trust boundaries diagram, threats → mitigations → residual risk.
- Reference architectures: zero trust, PKI, DevSecOps pipeline, cloud security posture, SIEM/SOC data flows.
- Control mapping where relevant: NIST CSF 2.0, CIS Controls v8, ISO 27001 Annex A, MITRE ATT&CK.

## Security-specific rules
- Defensive and architectural content only. No working exploit code, malware, or step-by-step attack instructions.
- Never describe real vulnerabilities in my employer's or any real organisation's environment.
- Tool configs and detection rules use fictional environments.

## Repo hygiene
- One Python venv: `.venv/`. (A stray `.venv2/` existed — must not be tracked.)

## Build
`mkdocs build --strict` must pass. Update `nav:` in `mkdocs.yml` for new pages.
