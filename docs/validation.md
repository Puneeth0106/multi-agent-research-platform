# Publication validation

The checks below describe local publication preparation, not a verified deployment.

## Completed

- Compiled the 13 Python application/dashboard files without importing them, avoiding AWS calls and embedding downloads.
- Checked both embedded JavaScript programs with `node --check`, including the actual HTML string returned by the dashboard function.
- Parsed the TensorZero TOML, resolved both prompt-template paths, and parsed their Jinja-compatible syntax. This does not validate the TensorZero runtime schema or full MiniJinja compatibility.
- Parsed the GitHub Actions YAML and confirmed a manual-only trigger. Checked `bootstrap.sh` and each workflow shell block with `bash -n` after substituting Actions expressions.
- Parsed the JSON secret example and `.env.example`; compared mandatory secret keys with `Config` and confirmed provider/shared-key placeholders.
- Parsed both requirements files as package requirement declarations; packages were not installed.
- Checked relative Markdown links, local heading anchors, and code-fence pairing. The supplied upstream source link was accessible through the web browsing tool. Localhost/example URLs require running services and were not probed.
- Scanned publication files for common AWS, GitHub, model-provider token, and private-key signatures; manually reviewed credential-bearing configuration. No apparent live credentials were found. This is not an exhaustive secret audit; a dedicated secret scanner was unavailable.
- Verified all 33 original bundle files against SHA-256 hashes after moving them to the ignored backup directory.
- Compared working source with the original bundle: only README, Git ignore/attributes, and the workflow trigger differ among copied files. New files provide examples, build-context exclusions, and documentation.

## Not tested or blocked

- Dependency installation/resolution, Python imports, running servers, browser interactions, PDF output, and Docker image builds.
- Redis, PostgreSQL/pgvector, TensorZero, OpenAI, Groq, Bedrock, Secrets Manager, LangSmith, PyRIT runtime, and end-to-end report generation.
- Terraform formatting/provider validation/planning/apply: Terraform is not installed. Infrastructure was inspected as text only. The Windows bootstrap was reviewed but not executed.
- Mermaid rendering: the diagram was reviewed as source only.
- Git initialization, Git's ignore/attribute evaluation, commits, and push: this workspace denies writes to `.git` (`Operation not permitted`). No prior local Git history or remotes were present.
- GitHub CLI cannot resolve/connect to `api.github.com`, and `gh auth status` reports invalid authentication. The target repository's existence, public visibility, metadata, and branch state could not be verified. No repository creation or push is claimed.

The ignored `.publication/` directory contains local check tooling and a continuation script. It is not part of the published project. The checker requires the locally available PyYAML, python-dotenv, Jinja2, and packaging modules plus Node and Bash; these are validation tools, not new application dependencies.
