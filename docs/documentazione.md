# SharpGuard — Documentazione Completa

> **Strumento di analisi della sicurezza per .NET**  
> SAST (Static Application Security Testing) + SCA (Software Composition Analysis)  
> Versione 1.0.0 | Licenza MIT | [https://github.com/Lupala-coder/SharpGuard](https://github.com/Lupala-coder/SharpGuard)

---

## Indice

1. [Panoramica](#1-panoramica)
2. [Installazione](#2-installazione)
3. [Quick Start](#3-quick-start)
4. [CLI — Utilizzo](#4-cli--utilizzo)
5. [NuGet Analyzer — Utilizzo](#5-nuget-analyzer--utilizzo)
6. [Regole SAST](#6-regole-sast)
7. [Software Composition Analysis (SCA)](#7-software-composition-analysis-sca)
8. [Formati di Report](#8-formati-di-report)
9. [Configurazione Severità](#9-configurazione-severità)
10. [Integrazione CI/CD](#10-integrazione-cicd)
11. [Come funziona](#11-come-funziona)
12. [Risoluzione Problemi](#12-risoluzione-problemi)
13. [Riferimenti](#13-riferimenti)

---

## 1. Panoramica

SharpGuard è uno strumento di analisi statica della sicurezza per applicazioni .NET. Analizza codice C# per vulnerabilità di sicurezza, correlandole a **CWE** (Common Weakness Enumeration) e alle **CVE** note (Common Vulnerabilities and Exposures).

### 1.1 Due pacchetti NuGet

SharpGuard si distribuisce in **due pacchetti NuGet separati** che condividono lo stesso motore di regole:

| Pacchetto NuGet | Tipo | Comando / Utilizzo | Scopo |
|-----------------|------|-------------------|-------|
| **`SharpGuard`** | `dotnet tool` | `SharpGuard analyze` | CLI standalone: SAST + SCA, report Console/JSON/SARIF |
| **`SharpGuard.Analyzers`** | Roslyn Analyzer | `<PackageReference>` | Build-time: warning/error in IDE e `dotnet build` |

**Principio chiave:** le regole SAST sono scritte **una sola volta** in `SharpGuard.Core` (`netstandard2.0`, zero I/O) e consumate da entrambi i pacchetti.

### 1.2 Funzionalità

- **20 regole SAST** — copertura OWASP Top 10, CWE Top 25
- **SCA via OSV.dev** — scansione dipendenze NuGet per CVE note, cache 24h
- **3 formati**: Console, JSON, **SARIF v2.1.0** (GitHub/Azure DevOps)
- **Anti-falsi-positivi**: `SemanticModel.GetSymbolInfo()`, mai match testuali
- **3 Code Fix** automatici in IDE
- **Configurabile**: `.editorconfig`, `#pragma`, soppressione selettiva

---

## 2. Installazione

### 2.1 Prerequisiti

- **.NET SDK 8.0** o superiore
- Connessione Internet (per SCA; opzionale con `--offline`)

### 2.2 CLI — `dotnet tool install`

```bash
# Installazione globale
dotnet tool install --global SharpGuard

# Verifica
SharpGuard --help

# Aggiornamento
dotnet tool update --global SharpGuard

# Disinstallazione
dotnet tool uninstall --global SharpGuard

# Installazione locale (repository-specific)
dotnet new tool-manifest
dotnet tool install SharpGuard
```

### 2.3 Analyzer — `PackageReference`

Nel `.csproj` del progetto da analizzare:

```xml
<ItemGroup>
  <PackageReference Include="SharpGuard.Analyzers" Version="1.0.0">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
  </PackageReference>
</ItemGroup>
```

Oppure:

```bash
dotnet add package SharpGuard.Analyzers
```

Dopo l'installazione, gli analizzatori si attivano automaticamente a ogni `dotnet build`.

---

## 3. Quick Start

```bash
# Prima analisi (output console)
SharpGuard analyze --path ./MySolution.sln

# Forma abbreviata
SharpGuard analyze -p ./MySolution.sln

# Con output SARIF per CI
SharpGuard analyze -p ./src -f sarif -o results.sarif

# CI mode: solo finding alti/critici, exit code 1 se presenti
SharpGuard analyze -p ./src -s high --fail-on-findings
```

---

## 4. CLI — Utilizzo

### 4.1 Sintassi

```bash
SharpGuard analyze [opzioni]
```

**Tutte le opzioni vanno DOPO `analyze`.**

### 4.2 Opzioni

| Opzione | Alias | Tipo | Default | Descrizione |
|---------|-------|------|---------|-------------|
| `--path` | `-p` | `string` | *(obbligatorio)* | Path a `.sln`, `.csproj`, o directory |
| `--format` | `-f` | `string` | `console` | `console`, `json`, `sarif` |
| `--output` | `-o` | `string` | stdout | File di output |
| `--severity-threshold` | `-s` | `string` | *(tutte)* | `critical`, `high`, `medium`, `low`, `info` |
| `--offline` | | `bool` | `false` | Salta rete SCA, solo cache |
| `--fail-on-findings` | | `bool` | `false` | Exit code 1 se finding presenti |

### 4.3 Exit Code

| Codice | Significato |
|--------|-------------|
| `0` | Successo, nessun finding sopra soglia |
| `1` | Finding trovati con `--fail-on-findings` |
| `2` | Errore durante l'analisi |
| `3` | Argomenti CLI non validi |

### 4.4 Esempi

```bash
# Base
SharpGuard analyze -p ./MySolution.sln

# JSON su file
SharpGuard analyze -p ./src -f json -o report.json

# SARIF per GitHub
SharpGuard analyze -p ./MySolution.sln -f sarif -o results.sarif

# Solo vulnerabilità critiche/alte
SharpGuard analyze -p ./src -s high --fail-on-findings

# Offline (nessuna chiamata HTTP)
SharpGuard analyze -p ./src --offline

# Singolo file .cs
SharpGuard analyze -p ./src/Services/AuthService.cs
```

### 4.5 Flusso di analisi

1. **Caricamento**: MSBuildWorkspace (con fallback AdhocWorkspace)
2. **SAST**: 20 regole ispezionano ogni syntax tree con `SemanticModel`
3. **SCA**: dipendenze NuGet → OSV.dev API → cache 24h
4. **Report**: aggregazione, filtro severità, writer del formato scelto

---

## 5. NuGet Analyzer — Utilizzo

### 5.1 Come funziona

Il pacchetto `SharpGuard.Analyzers` contiene 20 `DiagnosticAnalyzer` eseguiti dal compilatore Roslyn. Ogni analyzer è registrato per un `SyntaxKind` specifico, esecuzione **concurrente** (`EnableConcurrentExecution`) e **thread-safe**. I file generati automaticamente (`.g.cs`) vengono ignorati.

### 5.2 Mapping Severità

| SharpGuard | Roslyn DiagnosticSeverity | Effetto |
|------------|--------------------------|---------|
| `Critical` | `Error` | Blocca la compilazione |
| `High` | `Warning` | Warning giallo |
| `Medium` | `Warning` | Warning giallo |
| `Low` | `Info` | Messaggio informativo |
| `Info` | `Hidden` | Nascosto |

### 5.3 Code Fix (💡 in IDE)

| Code Fix | Regola | Azione |
|----------|--------|--------|
| Weak Cryptography | SG0006 | `MD5.Create()` → `SHA256.Create()`, `DES` → `Aes` |
| XXE | SG0008 | `DtdProcessing.Parse` → `Prohibit`, `XmlResolver` → `null` |
| Insecure Random | SG0007 | TODO comment per migrare a `RandomNumberGenerator` |

### 5.4 Disabilitazione selettiva

```csharp
// Per blocco di codice
#pragma warning disable SG0001
LegacyMethod();
#pragma warning restore SG0001

// Per metodo
[SuppressMessage("Security", "SG0001")]
public void LegacyMethod() { }
```

### 5.5 Performance

- `EnableConcurrentExecution()` — esecuzione parallela
- `ConfigureGeneratedCodeAnalysis(None)` — ignora file generati
- `SemanticModel` è cachato da Roslyn

---

## 6. Regole SAST

### 6.1 Elenco completo

| ID | Regola | CWE | Severità | Categoria |
|----|--------|-----|----------|-----------|
| SG0001 | SQL Injection | CWE-89 | High | Injection |
| SG0002 | Command Injection | CWE-78 | High | Injection |
| SG0003 | Path Traversal | CWE-22 | Medium | Injection |
| SG0004 | Insecure Deserialization | CWE-502 | Critical | Deserialization |
| SG0005 | Hardcoded Secrets | CWE-798 | Critical | InformationDisclosure |
| SG0006 | Weak Cryptography | CWE-327 | High | Cryptography |
| SG0007 | Insecure Random | CWE-338 | Medium | Cryptography |
| SG0008 | XXE | CWE-611 | High | Configuration |
| SG0009 | TLS Validation Disabled | CWE-295 | High | Configuration |
| SG0010 | Sensitive Data Logging | CWE-532 | Medium | Logging |
| SG0011 | LDAP Injection | CWE-90 | High | Injection |
| SG0012 | XPath Injection | CWE-643 | High | Injection |
| SG0013 | Cross-Site Scripting (XSS) | CWE-79 | High | Injection |
| SG0014 | Open Redirect | CWE-601 | Medium | Injection |
| SG0015 | SSRF | CWE-918 | High | Injection |
| SG0016 | Insecure Cookie | CWE-614 | Medium | Configuration |
| SG0017 | Weak RSA Key Size | CWE-326 | High | Cryptography |
| SG0018 | Missing Authorization | CWE-306 | Medium | Configuration |
| SG0019 | Information Exposure | CWE-209 | Medium | InformationDisclosure |
| SG0020 | TOCTOU Race Condition | CWE-367 | Low | Injection |

### 6.2 Dettaglio (esempi per ogni regola)

#### SG0001 — SQL Injection (CWE-89)

```csharp
// 🚫 Vulnerabile
var cmd = new SqlCommand("SELECT * FROM Users WHERE Id = " + userId, conn);
cmd.ExecuteReader();

// ✅ Sicuro
var cmd = new SqlCommand("SELECT * FROM Users WHERE Id = @Id", conn);
cmd.Parameters.AddWithValue("@Id", userId);
cmd.ExecuteReader();
```

#### SG0002 — Command Injection (CWE-78)

```csharp
// 🚫 Vulnerabile
Process.Start(userInput);

// ✅ Sicuro
Process.Start("notepad.exe", "readme.txt");
```

#### SG0003 — Path Traversal (CWE-22)

```csharp
// 🚫 Vulnerabile
var content = File.ReadAllText(userInput);

// ✅ Sicuro
var safePath = Path.GetFullPath(Path.Combine(baseDir, Sanitize(userInput)));
if (!safePath.StartsWith(baseDir)) throw new SecurityException();
var content = File.ReadAllText(safePath);
```

#### SG0004 — Insecure Deserialization (CWE-502)

```csharp
// 🚫 Vulnerabile
var formatter = new BinaryFormatter();
var obj = formatter.Deserialize(stream);

// ✅ Sicuro
var obj = JsonSerializer.Deserialize<MyType>(stream);
```

#### SG0005 — Hardcoded Secrets (CWE-798)

```csharp
// 🚫 Vulnerabile
var password = "MySuperSecret123!";
var connStr = "Server=prod;User=sa;Password=P@ssw0rd!";

// ✅ Sicuro
var password = config["Secrets:DbPassword"];
```

#### SG0006 — Weak Cryptography (CWE-327)

```csharp
// 🚫 Vulnerabile — Code Fix: sostituisce con SHA256/Aes
var md5 = MD5.Create();
var des = DES.Create();

// ✅ Sicuro
var sha256 = SHA256.Create();
var aes = Aes.Create();
```

#### SG0007 — Insecure Random (CWE-338)

```csharp
// 🚫 Vulnerabile
var rng = new Random();

// ✅ Sicuro
var bytes = new byte[32];
RandomNumberGenerator.Fill(bytes);
```

#### SG0008 — XXE (CWE-611)

```csharp
// 🚫 Vulnerabile — Code Fix: imposta Prohibit / null
var settings = new XmlReaderSettings { DtdProcessing = DtdProcessing.Parse };
var doc = new XmlDocument { XmlResolver = new XmlUrlResolver() };

// ✅ Sicuro
var settings = new XmlReaderSettings { DtdProcessing = DtdProcessing.Prohibit };
```

#### SG0009 — TLS Validation Disabled (CWE-295)

```csharp
// 🚫 Vulnerabile
ServicePointManager.ServerCertificateValidationCallback = (_, _, _, _) => true;

// ✅ Sicuro — rimuovere l'override
```

#### SG0010 — Sensitive Data Logging (CWE-532)

```csharp
// 🚫 Vulnerabile
logger.LogInformation("Password: {pw}", password);

// ✅ Sicuro
logger.LogInformation("User {UserId} authenticated", userId);
```

#### SG0011 — LDAP Injection (CWE-90)

```csharp
// 🚫 Vulnerabile
var searcher = new DirectorySearcher("(uid=" + userName + ")");

// ✅ Sicuro
var searcher = new DirectorySearcher($"(uid={EscapeLdapFilter(userName)})");
```

#### SG0012 — XPath Injection (CWE-643)

```csharp
// 🚫 Vulnerabile
navigator.Select("//user[name='" + userName + "']");

// ✅ Sicuro — parametrizzato
```

#### SG0013 — XSS (CWE-79)

```csharp
// 🚫 Vulnerabile
Response.Write("<div>" + userInput + "</div>");

// ✅ Sicuro
Response.Write("<div>" + HttpUtility.HtmlEncode(userInput) + "</div>");
```

#### SG0014 — Open Redirect (CWE-601)

```csharp
// 🚫 Vulnerabile
return Redirect(returnUrl);

// ✅ Sicuro
return LocalRedirect(returnUrl);  // Solo URL locali
```

#### SG0015 — SSRF (CWE-918)

```csharp
// 🚫 Vulnerabile
var result = await client.GetStringAsync(userUrl);

// ✅ Sicuro — whitelist domini consentiti
if (!IsAllowedHost(new Uri(userUrl))) throw new SecurityException();
```

#### SG0016 — Insecure Cookie (CWE-614)

```csharp
// 🚫 Vulnerabile
var cookie = new HttpCookie("auth", token);

// ✅ Sicuro
cookie.HttpOnly = true;
cookie.Secure = true;
cookie.SameSite = SameSiteMode.Strict;
```

#### SG0017 — Weak RSA Key (CWE-326)

```csharp
// 🚫 Vulnerabile
using var rsa = RSA.Create(1024);

// ✅ Sicuro
using var rsa = RSA.Create(2048);  // Minimo
```

#### SG0018 — Missing Authorization (CWE-306)

```csharp
// 🚫 Vulnerabile
public class AdminController : Controller { }

// ✅ Sicuro
[Authorize(Roles = "Admin")]
public class AdminController : Controller { }
```

#### SG0019 — Information Exposure (CWE-209)

```csharp
// 🚫 Vulnerabile
catch (Exception ex) { return BadRequest(ex.Message); }

// ✅ Sicuro
catch (Exception ex) {
    _logger.LogError(ex, "Operation failed");
    return BadRequest("An error occurred.");
}
```

#### SG0020 — TOCTOU (CWE-367)

```csharp
// 🚫 Vulnerabile — race condition
if (File.Exists(path)) {
    var content = File.ReadAllText(path);
}

// ✅ Sicuro
try {
    var content = File.ReadAllText(path);
} catch (FileNotFoundException) { }
```

---

## 7. Software Composition Analysis (SCA)

### 7.1 Flusso

```
Progetto (.csproj/.sln)
  → Estrai PackageReference (XDocument)
    → Cache? (~/.sharpguard/cache/, TTL 24h)
      → Cache valida: usa dati locali
      → Non in cache/scaduta: POST https://api.osv.dev/v1/query
        → Salva in cache
  → List<VulnerabilityResult>
```

### 7.2 Gestione errori

| Scenario | Comportamento |
|----------|---------------|
| `429` Rate Limit | Retry ×3 con backoff esponenziale |
| `5xx` Server Error | Skip pacchetto, continua |
| Timeout 10s | Usa cache se disponibile |
| `--offline` | Solo cache, zero HTTP |
| Cache corrotta | Elimina + re-fetch |

### 7.3 Limitazioni

- **Solo CLI**: l'Analyzer NuGet non può fare I/O di rete/filesystem
- **Solo NuGet**: ecosistema `NuGet`, altri formati non analizzati
- **Dipendenze dirette**: non vengono risolte le transitive

---

## 8. Formati di Report

Pattern **Strategy** — 3 implementazioni.

| Formato | Opzione | Uso |
|---------|---------|-----|
| **Console** | `-f console` (default) | Output umano, raggruppato per severità |
| **JSON** | `-f json` | Machine-readable, post-processing |
| **SARIF v2.1.0** | `-f sarif` | GitHub Code Scanning, Azure DevOps |

### Mapping SARIF

| SharpGuard | SARIF `level` |
|------------|---------------|
| Critical | `error` |
| High | `error` |
| Medium | `warning` |
| Low | `note` |
| Info | `none` |

---

## 9. Configurazione Severità

### 9.1 Via `.editorconfig`

```ini
# Root del repository
[*.cs]

# SQL injection come errore bloccante
dotnet_diagnostic.SG0001.severity = error

# Disabilita nei test
[**/Tests/**.cs]
dotnet_diagnostic.SG0010.severity = none
dotnet_diagnostic.SG0005.severity = none
```

### 9.2 Valori

| Valore | Effetto |
|--------|---------|
| `error` | Blocca build |
| `warning` | Warning |
| `suggestion` | Suggerimento IDE |
| `silent` | Silenzioso |
| `none` | Disabilitato |

### 9.3 Via `#pragma` (solo Analyzer)

```csharp
#pragma warning disable SG0005
var testPassword = "test";
#pragma warning restore SG0005
```

### 9.4 TreatWarningsAsErrors

Per l'Analyzer, in `.csproj`:

```xml
<PropertyGroup>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  <WarningsAsErrors>SG0001;SG0004;SG0005;SG0006;SG0008;SG0009;SG0017</WarningsAsErrors>
</PropertyGroup>
```

---

## 10. Integrazione CI/CD

### 10.1 GitHub Actions

```yaml
name: Security Analysis
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  sharpguard:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'
      - run: dotnet restore
      - run: dotnet tool install --global SharpGuard
      - run: |
          SharpGuard analyze \
            --path ./src \
            --format sarif \
            --severity-threshold high \
            --output sharpguard.sarif \
            --fail-on-findings
      - uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: sharpguard.sarif
```

### 10.2 Azure DevOps

```yaml
steps:
  - task: UseDotNet@2
    inputs: { version: '8.0.x' }
  - script: dotnet tool install --global SharpGuard
  - script: |
      SharpGuard analyze \
        --path $(Build.SourcesDirectory)/src \
        --format sarif \
        --output $(Build.ArtifactStagingDirectory)/sharpguard.sarif \
        --severity-threshold high \
        --fail-on-findings
  - task: PublishBuildArtifacts@1
    condition: always()
    inputs:
      pathToPublish: '$(Build.ArtifactStagingDirectory)/sharpguard.sarif'
```

### 10.3 GitLab CI

```yaml
sharpguard:
  stage: test
  image: mcr.microsoft.com/dotnet/sdk:8.0
  script:
    - dotnet tool install --global SharpGuard
    - SharpGuard analyze -p ./src -f sarif -o sharpguard.sarif -s high --fail-on-findings
  artifacts:
    paths: [sharpguard.sarif]
    when: always
```

---

## 11. Come funziona

### 11.1 Motore di regole condiviso

SharpGuard usa un motore di regole **unico** che alimenta entrambe le modalità (CLI e Analyzer). Ogni regola SAST:

1. È scritta una volta sola — nessuna duplicazione tra CLI e Analyzer
2. Usa il compilatore Roslyn per analizzare il codice (non match testuali)
3. Verifica il **contesto d'uso** tramite l'API `SemanticModel`: risolve tipi, metodi, gerarchie di classi
4. Produce un finding solo quando la combinazione di API chiamata + argomenti non costanti indica una reale vulnerabilità

### 11.2 Flusso di analisi

```
Codice sorgente (.cs)
        │
        ▼
Compilatore Roslyn ──→ SyntaxTree + SemanticModel
        │
        ▼
20 regole SAST ──────→ Ogni regola cerca il suo pattern specifico
        │               (es. SG0001 cerca SqlCommand con concatenazione)
        ▼
Finding ─────────────→ Severità, CWE, file, linea, messaggio, remediation
        │
        ▼
Report ──────────────→ Console / JSON / SARIF
```

### 11.3 SAST vs SCA

| | SAST | SCA |
|---|------|-----|
| **Cosa analizza** | Codice sorgente C# | Dipendenze NuGet (PackageReference) |
| **Cosa rileva** | Pattern di vulnerabilità nel codice | CVE note nei pacchetti |
| **Fonte dati** | Compilatore Roslyn | API OSV.dev |
| **Disponibile in CLI** | ✅ | ✅ |
| **Disponibile in Analyzer** | ✅ | ❌ (richiede rete/filesystem) |

### 11.4 Perché pochi falsi positivi

1. **Risoluzione simboli**: ogni metodo, tipo e proprietà viene risolto tramite il compilatore. Una chiamata a `.ExecuteReader()` viene segnalata solo se appartiene a un tipo che implementa `IDbCommand`.
2. **Analisi del contesto**: una stringa che contiene "password" non basta — deve essere assegnata a una variabile con nome sensibile, o passata a un costruttore con parametro sospetto.
3. **Best effort**: se un file non compila o un nodo non è analizzabile, viene saltato senza bloccare l'analisi.

### 11.5 Performance

- **CLI**: analisi di una solution media (~50 progetti) in pochi secondi
- **Analyzer**: esecuzione parallela (`EnableConcurrentExecution`), file generati ignorati
- **SCA**: cache locale 24h per evitare chiamate ripetute all'API OSV.dev

---

## 12. Risoluzione Problemi

### MSBuild non disponibile

**Sintomo:** `[Info] MSBuild not available; using file-based analysis`

**Causa:** MSBuild non installato. L'analisi prosegue in modalità AdhocWorkspace (senza risoluzione NuGet).

**Soluzione:** installare .NET SDK 8.0 con workload MSBuild, o su macOS `brew install mono`.

### Progetto non compilabile

1. `dotnet restore` prima dell'analisi
2. `dotnet build` per verificare che compili
3. Per file `.cs` singoli: passa direttamente il percorso

### Rate limiting OSV.dev

Usa `--offline` dopo la prima esecuzione. Cache in `~/.sharpguard/cache/` con TTL 24h.

### L'Analyzer non produce warning

1. Verifica `PrivateAssets` e `IncludeAssets` nel `PackageReference`
2. Controlla `.editorconfig` — potresti avere `severity = none`
3. `dotnet clean && dotnet build`

---

## 13. Riferimenti

### Pacchetti NuGet

| Pacchetto | Versione | Tipo |
|-----------|----------|------|
| `SharpGuard` | 1.0.0 | `dotnet tool` |
| `SharpGuard.Analyzers` | 1.0.0 | Roslyn Analyzer |

### Metadata

| Campo | Valore |
|-------|--------|
| Package ID (CLI) | `SharpGuard` |
| Package ID (Analyzer) | `SharpGuard.Analyzers` |
| Versione | `1.0.0` |
| Autore | Paolino Salmone |
| Licenza | MIT |
| Repository | [github.com/Lupala-coder/SharpGuard](https://github.com/Lupala-coder/SharpGuard) |
| Sito web | [lupala.com](https://lupala.com) |

### Link esterni

- [CWE Database](https://cwe.mitre.org)
- [OSV.dev API](https://osv.dev/docs)
- [SARIF v2.1.0](https://docs.oasis-open.org/sarif/sarif/v2.1.0)
- [Roslyn Analyzer Docs](https://learn.microsoft.com/dotnet/csharp/roslyn-sdk)

---

## Licenza

MIT — vedi [LICENSE](https://github.com/Lupala-coder/SharpGuard/blob/main/LICENSE).

## Autore

**Paolino Salmone** — [https://www.lupala.com](https://www.lupala.com)
