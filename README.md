# SharpGuard

.NET security analysis tool — **SAST** (Static Application Security Testing) + **SCA** (Software Composition Analysis).

Analizza codice C# per vulnerabilità di sicurezza, correlandole a **CWE** e **CVE**. Distribuibile in due modalità che condividono lo stesso motore di regole.

## Caratteristiche

- **10 regole SAST** che coprono le vulnerabilità OWASP Top 10 (SQL injection, XXE, deserializzazione insicura, crittografia debole, ecc.)
- **SCA via OSV.dev** — scansione delle dipendenze NuGet per CVE note
- **Due modalità di deployment:**
  - **CLI standalone** — analisi completa con report Console/JSON/SARIF
  - **Roslyn Analyzer** — warning/error a tempo di compilazione direttamente in IDE e `dotnet build`
- **SARIF v2.1.0** per integrazione con GitHub Code Scanning e Azure DevOps
- **Regole condivise** — la logica di detection è scritta una volta sola in `SharpGuard.Core` e consumata sia dal CLI che dall'Analyzer

## Quick Start

### CLI Tool

```bash
# Installa come dotnet global tool
dotnet tool install --global SharpGuard

# Analizza una soluzione
SharpGuard analyze --path ./MySolution.sln --format sarif --output results.sarif

# Analizza con soglia di severità e fail-on-findings per CI
SharpGuard analyze --path ./src --severity-threshold high --fail-on-findings
```

### Roslyn Analyzer

Aggiungi al `.csproj`:

```xml
<PackageReference Include="SharpGuard.Analyzers" Version="1.0.0">
  <PrivateAssets>all</PrivateAssets>
  <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
</PackageReference>
```

I warning appariranno automaticamente in `dotnet build`, Visual Studio e Rider.

## Opzioni CLI

| Opzione | Default | Descrizione |
|---------|---------|-------------|
| `--path`, `-p` | *(required)* | Percorso a `.sln`, `.csproj` o directory |
| `--format`, `-f` | `console` | Formato output: `console`, `json`, `sarif` |
| `--output`, `-o` | stdout | File di output |
| `--severity-threshold`, `-s` | *(tutte)* | Soglia minima: `critical`, `high`, `medium`, `low`, `info` |
| `--offline` | `false` | Salta le chiamate di rete SCA (usa solo cache) |
| `--fail-on-findings` | `false` | Exit code 1 se ci sono finding |

## Regole SAST

| ID | Nome | CWE | Severità |
|----|------|-----|----------|
| SG0001 | SQL Injection | CWE-89 | High |
| SG0002 | Command Injection | CWE-78 | High |
| SG0003 | Path Traversal | CWE-22 | Medium |
| SG0004 | Insecure Deserialization | CWE-502 | Critical |
| SG0005 | Hardcoded Secrets | CWE-798 | Critical |
| SG0006 | Weak Cryptography | CWE-327 | High |
| SG0007 | Insecure Random | CWE-338 | Medium |
| SG0008 | XXE | CWE-611 | High |
| SG0009 | TLS Validation Disabled | CWE-295 | High |
| SG0010 | Sensitive Data Logging | CWE-532 | Medium |

## Configurazione Severità (.editorconfig)

La severità di ogni regola è configurabile via `.editorconfig`:

```ini
[*.cs]
# Tratta SQL injection come error
dotnet_diagnostic.SG0001.severity = error

# Disabilita la regola per i test
dotnet_diagnostic.SG0010.severity = none
```

## Integrazione CI

### GitHub Actions

```yaml
- name: Run SharpGuard
  run: |
    dotnet tool install --global SharpGuard
    SharpGuard analyze --path ./src --format sarif --output sharpguard.sarif --fail-on-findings

- name: Upload SARIF
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: sharpguard.sarif
```

### Azure DevOps

```yaml
- task: DotNetCoreCLI@2
  inputs:
    command: custom
    custom: tool
    arguments: 'install --global SharpGuard'

- script: SharpGuard analyze --path $(Build.SourcesDirectory) --format sarif --output $(Build.ArtifactStagingDirectory)/sharpguard.sarif
```

## Build da Sorgente

```bash
# Prerequisiti: .NET 8 SDK

# Clona
git clone https://github.com/Lupala-coder/SharpGuard.git
cd SharpGuard

# Build
dotnet build SharpGuard.sln

# Test (37 test)
dotnet test SharpGuard.sln

# Pack CLI tool
dotnet pack src/SharpGuard.Cli/SharpGuard.Cli.csproj -c Release

# Pack Analyzer NuGet
dotnet pack src/SharpGuard.Analyzers/SharpGuard.Analyzers.csproj -c Release
```

## Architettura

```
src/
  SharpGuard.Core         — Astrazioni (ISecurityRule) e 10 regole SAST. Zero I/O.
  SharpGuard.Analyzers    — DiagnosticAnalyzer + CodeFixProvider. Packaging NuGet.
  SharpGuard.Sca          — Client OSV.dev + cache. Solo lato CLI.
  SharpGuard.Reporting    — Reporter Console/JSON/SARIF (pattern Strategy).
  SharpGuard.Cli          — Front-end CLI con System.CommandLine.

tests/
  SharpGuard.Core.Tests        — Test per ogni regola (25 test)
  SharpGuard.Analyzers.Tests   — Test analyzer (6 test)
  SharpGuard.Sca.Tests         — Test SCA (6 test)
```

### Come Core viene condiviso tra CLI e Analyzer

`SharpGuard.Core` (netstandard2.0) contiene `ISecurityRule`, `SecurityRuleBase` e tutte le 10 regole. Non fa I/O di rete né filesystem — solo analisi Roslyn. Sia il CLI che l'Analyzer referenziano questo progetto e chiamano `RuleFactory.CreateAll()` per ottenere le regole.

### Perché la SCA resta fuori dall'Analyzer

L'SCA richiede chiamate HTTP a `api.osv.dev`, parsing di file `.csproj`, e cache su filesystem. Un Roslyn Analyzer deve essere veloce, deterministico, concurrent-safe e senza I/O di rete. La SCA è disponibile solo nel CLI.

### Scelte anti-falsi-positivi

- **Mai match testuali**: ogni regola usa `SemanticModel.GetSymbolInfo()` per risolvere tipi e metodi
- **Analisi del contesto**: non basta trovare una stringa — verifichiamo che sia usata in un contesto sensibile (es. `SqlCommand`, `Process.Start`)
- **Best effort**: progetti non compilabili vengono analizzati al meglio, con skip dei nodi problematici

## Licenza

MIT — vedi [LICENSE](LICENSE).

## Autore

Paolino Salmone — [https://lupala.com](https://lupala.com)
