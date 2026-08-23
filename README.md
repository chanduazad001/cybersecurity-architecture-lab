# Cybersecurity Architecture Lab

Application, cloud, and enterprise security architecture — documented as
deliberate architecture practice: programs, controls, trade-offs, and the
reasoning behind security decisions.

**Published site:** https://chanduazad001.github.io/cybersecurity-architecture-lab/

Part of a wider set of labs linked from
[chanduazad001.github.io](https://chanduazad001.github.io/), including
[Architecture Lab](https://chanduazad001.github.io/architecture-lab/)
(cloud architecture, platform engineering, and system design).

## Structure

Each security domain lives under `docs/<domain>/` as its own top-level nav
section, sibling to the others (e.g. Vulnerability Management, and future
domains such as Application Security, Cloud Security, IAM, and Threat
Modeling). Within a domain, pages follow a consistent set appropriate to
that domain — typically an Overview, the program/process detail, and a
Decisions page recording ADRs.

Diagrams are written as [Mermaid](https://mermaid.js.org/) inline in the
Markdown, so they stay version-controlled alongside the text.

## Local development

```bash
python -m venv .venv
.venv\Scripts\activate      # Windows
source .venv/bin/activate   # macOS/Linux

pip install -r requirements.txt

mkdocs serve
```

Then open http://127.0.0.1:8000.

## Publishing

Pushes to `master` that touch `docs/`, `mkdocs.yml`, or `requirements.txt`
trigger `.github/workflows/docs.yml`, which builds the site with
`mkdocs build --strict` and deploys it to GitHub Pages.

## License

See [LICENSE](LICENSE).
