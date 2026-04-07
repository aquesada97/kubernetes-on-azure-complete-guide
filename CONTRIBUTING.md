# Contributing to AKS Training

Thank you for considering contributing to the AKS Training repository! Contributions are what make this resource valuable for the community.

---

## 📋 Table of Contents

- [Code of Conduct](#code-of-conduct)
- [How to Contribute](#how-to-contribute)
- [Style Guide](#style-guide)
- [Pull Request Process](#pull-request-process)
- [Reporting Issues](#reporting-issues)

---

## Code of Conduct

All contributors are expected to treat others with respect and professionalism. Harassment, discrimination, or abusive behavior will not be tolerated.

---

## How to Contribute

### Types of Contributions

- **Bug fixes** — Correct errors in commands, YAML, or explanations
- **Content improvements** — Improve clarity, add missing detail, update outdated information
- **New content** — Add new labs, modules, or deep-dive documents
- **Translations** — Translate content to other languages

### Getting Started

1. **Fork** the repository on GitHub
2. **Clone** your fork:
   ```bash
   git clone https://github.com/YOUR-USERNAME/aks-training.git
   cd aks-training
   ```
3. **Create a branch** for your changes:
   ```bash
   git checkout -b feature/improve-rbac-section
   ```
4. Make your changes
5. **Commit** your changes (see commit message guidelines below)
6. **Push** to your fork:
   ```bash
   git push origin feature/improve-rbac-section
   ```
7. Open a **Pull Request**

### Commit Message Guidelines

Use the following format:
```
<type>: <short description>

[optional body]
```

Types:
- `fix` — Bug fix or correction
- `feat` — New content or feature
- `docs` — Documentation updates
- `style` — Formatting only (no content change)
- `refactor` — Restructuring without content change

Examples:
```
fix: correct az aks create flag in 02-getting-started
feat: add lab for Azure Workload Identity
docs: update prerequisites with Helm 3.13 instructions
```

---

## Style Guide

### Markdown Conventions

- Use `#` for page title (only one per file)
- Use `##` for major sections
- Use `###` for subsections
- Use `####` for sub-subsections
- Always include a blank line before and after headings and code blocks

### Code Blocks

Always specify the language for syntax highlighting:

````markdown
```bash
az aks create --name myCluster --resource-group myRG
```
````

````markdown
```yaml
apiVersion: apps/v1
kind: Deployment
```
````

````markdown
```powershell
New-AzResourceGroup -Name myRG -Location eastus
```
````

### Commands and Flags

When explaining a command with multiple flags, break it across lines using `\` (bash) or `` ` `` (PowerShell) and add inline comments:

```bash
az aks create \
  --name myCluster \          # Cluster name
  --resource-group myRG \     # Resource group
  --node-count 3              # Initial node count
```

### YAML Files

- Always include `apiVersion`, `kind`, `metadata`, and `spec`
- Add comments to explain non-obvious fields
- Use 2-space indentation
- Include resource `requests` and `limits` in all Pod specs

### Tables

Use tables for comparisons, option lists, and reference data:

```markdown
| Option | Description | Default |
|--------|-------------|---------|
| `--node-count` | Number of nodes | 3 |
```

### Admonitions / Callouts

Use blockquotes with emoji for notes, warnings, and tips:

```markdown
> 💡 **Tip:** Use `--generate-ssh-keys` to automatically create SSH keys.

> ⚠️ **Warning:** Deleting a node pool is irreversible.

> 📝 **Note:** This feature requires AKS 1.24 or later.
```

---

## Pull Request Process

1. Ensure your PR targets the `main` branch
2. Fill out the PR template completely
3. Link any related issues using `Fixes #123` or `Relates to #456`
4. Request review from at least one maintainer
5. Address all review comments before merging
6. Squash commits if requested by a maintainer

### PR Checklist

Before submitting, verify:

- [ ] All commands have been tested
- [ ] YAML manifests are valid (use `kubectl apply --dry-run=client`)
- [ ] Markdown renders correctly (use a local preview tool)
- [ ] No sensitive information (subscription IDs, credentials) included
- [ ] Links are not broken
- [ ] Code blocks have language identifiers

---

## Reporting Issues

If you find an error or have a suggestion:

1. Check existing [Issues](https://github.com/your-org/aks-training/issues) to avoid duplicates
2. Open a new issue with:
   - **Clear title** describing the problem
   - **File path** where the issue exists
   - **Description** of the problem and expected behavior
   - **Suggested fix** (optional)

---

## Questions?

Open a [Discussion](https://github.com/your-org/aks-training/discussions) for general questions or learning discussions.
