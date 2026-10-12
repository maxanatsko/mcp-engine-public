# Security Policy

This policy covers SemanticOps MCP (formerly MCP Engine) in every distribution: the Windows and macOS binaries, the `.mcpb` Claude Desktop extensions, the npm packages and the Visual Studio Code extension. It also covers SemanticOps Test Runner and the agent skills in this repository.

## Reporting a vulnerability

Report privately through either channel:

- **GitHub:** [Report a vulnerability](https://github.com/maxanatsko/mcp-engine-public/security/advisories/new) (private vulnerability reporting on this repository)
- **Email:** security@semanticops.dev

Please do not open a public issue, discussion or pull request for a suspected vulnerability.

A useful report includes:

- the product, version and variant (for example `mcp-engine-win-x64.exe` 3.9.119, read-only build)
- the operating system and the MCP client used
- steps to reproduce, and what an attacker gains
- whether you know of it being exploited

Do not include live credentials, tokens, connection strings or customer model data. If a reproduction needs a model, a minimal synthetic one is enough.

## What to expect

| Step | Target |
|---|---|
| Acknowledge your report | 3 business days |
| Initial assessment and severity | 10 business days |
| Fix or mitigation for critical and high severity | as fast as possible, aiming for 30 days |
| Fix for medium and low severity | next scheduled release |

We will keep you informed while we work on it, agree a disclosure date with you, and credit you in the advisory unless you prefer not to be named. These are targets, not guarantees.

Actively exploited vulnerabilities are reported to the authorities as the EU Cyber Resilience Act requires.

## Supported versions

Security fixes are released in the **latest stable release** only. There are no backports to older versions; update to the latest release to receive fixes.

| Product | Supported |
|---|---|
| SemanticOps MCP | latest stable release on the [releases page](https://github.com/maxanatsko/mcp-engine-public/releases) |
| SemanticOps Test Runner | latest stable `test-runner-v*` release |
| VS Code extension | latest Marketplace version |

SemanticOps MCP and Test Runner check GitHub for a newer stable release and tell you when one is available. The [privacy policy](https://www.semanticops.dev/privacy) explains what that check sends and how to disable it.

## Verifying releases

Each SemanticOps MCP release includes:

- `semanticops-mcp-release-assets.sha256`: SHA-256 digests of the release files
- `semanticops-mcp-release-provenance.json`: a machine-readable manifest of the release assets

Windows binaries are code-signed. macOS binaries are signed with an Apple Developer ID.

A CycloneDX software bill of materials (SBOM) is produced for every release. To request it, email security@semanticops.dev.

## Scope

In scope: the software listed above and its release artifacts.

Out of scope: Power BI, Analysis Services, Microsoft Entra and other Microsoft services; your MCP client or AI provider; the semanticops.dev website (report website issues to the same address, but they are not product vulnerabilities); and findings that need an already-compromised machine or account.

SemanticOps runs locally and acts with the permissions of the user and the credentials they supply. An AI agent doing something the user's settings allowed is not a vulnerability. Bypassing read-only or browse-only mode, a policy rule, masking or redaction, or the audit log is.

## Contact

Maksim Anatska, trading as Maxim Anatsko · security@semanticops.dev
