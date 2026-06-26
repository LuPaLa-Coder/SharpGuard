# SharpGuard — Documentazione completa

> Strumento di analisi della sicurezza per .NET: **SAST** (Static Application Security Testing) + **SCA** (Software Composition Analysis).

---

## Indice

1. [Panoramica](#1-panoramica)
2. [Installazione](#2-installazione)
3. [Utilizzo CLI](#3-utilizzo-cli)
4. [Utilizzo NuGet Analyzer](#4-utilizzo-nuget-analyzer)
5. [Regole SAST](#5-regole-sast)
6. [Software Composition Analysis (SCA)](#6-software-composition-analysis-sca)
7. [Formati di report](#7-formati-di-report)
8. [Configurazione severità (.editorconfig)](#8-configurazione-severità-editorconfig)
9. [Integrazione CI/CD](#9-integrazione-cicd)

---

## 1. Panoramica

SharpGuard analizza codice C# per vulnerabilità di sicurezza, correlandole a **CWE** (Common Weakness Enumeration) e **CVE** (Common Vulnerabilities and Exposures). Si distribuisce in due modalità:

| Modalità | Tipo | Target |
|----------|------|--------|
| **CLI standalone** | `dotnet tool` | Analisi completa con report Console/JSON/SARIF |
| **NuGet Analyzer** | Pacchetto NuGet (`analyzers/dotnet/cs`) | Warning/error a tempo di compilazione in IDE e `dotnet build` |

Entrambe le modalità condividono lo **stesso motore di regole**.

### Funzionalità principali

- **20 regole SAST** che coprono le vulnerabilità OWASP Top 10 e oltre
- **SCA via OSV.dev** — scansione dipendenze NuGet per CVE note
- **SARIF v2.1.0** per GitHub Code Scanning e Azure DevOps
- **Anti-falsi-positivi**: ogni regola verifica il contesto d'uso, non cerca semplici pattern testuali
- **Cache SCA** persistente su filesystem (`~/.sharpguard/cache/`) con TTL 24h

---

## 2. Installazione

### 2.1 CLI (dotnet tool)

```bash
# Installazione globale
dotnet tool install --global SharpGuard

# Verifica
SharpGuard --help

# Aggiornamento
dotnet tool update --global SharpGuard

# Disinstallazione
dotnet tool uninstall --global SharpGuard
```

### 2.2 NuGet Analyzer

Aggiungi il pacchetto al progetto:

```xml
<PackageReference Include="SharpGuard.Analyzers" Version="1.0.0">
  <PrivateAssets>all</PrivateAssets>
  <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
</PackageReference>
```

Oppure con `dotnet`:

```bash
dotnet add package SharpGuard.Analyzers
```

Dopo l'installazione, gli analizzatori vengono eseguiti automaticamente a ogni compilazione (`dotnet build`, Visual Studio, Rider).

---

## 3. Utilizzo CLI

### 3.1 Sintassi

```bash
SharpGuard [opzioni]
```

### 3.2 Opzioni

| Opzione | Alias | Default | Descrizione |
|---------|-------|---------|-------------|
| `--path` | `-p` | *(obbligatorio)* | Percorso a `.sln`, `.csproj` o directory |
| `--format` | `-f` | `console` | Formato output: `console`, `json`, `sarif` |
| `--output` | `-o` | stdout | File di output |
| `--severity-threshold` | `-s` | *(tutte)* | Soglia minima: `critical`, `high`, `medium`, `low`, `info` |
| `--offline` | — | `false` | Salta chiamate di rete SCA (usa solo cache) |
| `--fail-on-findings` | — | `false` | Exit code 1 se ci sono finding |

### 3.3 Codici di uscita

| Codice | Significato |
|--------|-------------|
| `0` | Successo, nessun finding |
| `1` | Finding trovati con `--fail-on-findings` |
| `2` | Errore durante l'analisi |
| `3` | Argomenti CLI non validi |

### 3.4 Esempi

```bash
# Analisi base con output console
SharpGuard -p ./MyApp.sln

# Output JSON su file
SharpGuard -p ./src -f json -o report.json

# Output SARIF per GitHub Code Scanning
SharpGuard -p ./MyApp.sln -f sarif -o results.sarif

# Solo vulnerabilità critiche/alte, fail in CI
SharpGuard -p ./src -s high --fail-on-findings

# Modalità offline (senza chiamate HTTP)
SharpGuard -p ./src --offline

# Analisi di un singolo progetto
SharpGuard -p ./src/MyApp/MyApp.csproj
```

---

## 4. Utilizzo NuGet Analyzer

### 4.1 Installazione

Vedi [sezione 2.2](#22-nuget-analyzer).

### 4.2 Comportamento

A ogni compilazione, SharpGuard analizza il progetto corrente e produce:

- **Errori** per vulnerabilità `Critical`
- **Warning** per vulnerabilità `High` e `Medium`
- **Info** per vulnerabilità `Low`
- **Hidden** per vulnerabilità `Info`

### 4.3 Code Fix

Tre code fix sono disponibili automaticamente in IDE:

| Regola | Fix |
|--------|-----|
| **SG0006** (Weak Cryptography) | Sostituisce `MD5.Create()` → `SHA256.Create()`, `DES.Create()` → `Aes.Create()`, ecc. |
| **SG0008** (XXE) | Imposta `DtdProcessing.Prohibit` o `XmlResolver = null` |
| **SG0007** (Insecure Random) | Aggiunge commento TODO per migrare a `RandomNumberGenerator` |

### 4.4 Disabilitazione selettiva

Per escludere un file specifico:

```csharp
#pragma warning disable SG0001
// codice da escludere
#pragma warning restore SG0001
```

Per disabilitare una regola su tutto il progetto, vedi [sezione 8](#8-configurazione-severità-editorconfig).

---

## 5. Regole SAST

### 5.1 Elenco completo

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
| SG0015 | Server-Side Request Forgery (SSRF) | CWE-918 | High | Injection |
| SG0016 | Insecure Cookie Configuration | CWE-614 | Medium | Configuration |
| SG0017 | Weak RSA Key Size | CWE-326 | High | Cryptography |
| SG0018 | Missing Authorization | CWE-306 | Medium | Configuration |
| SG0019 | Information Exposure via Error Messages | CWE-209 | Medium | InformationDisclosure |
| SG0020 | TOCTOU Race Condition | CWE-367 | Low | Injection |

### 5.2 Dettaglio regole

#### SG0001 — SQL Injection

- **Cosa rileva**: chiamate a `ExecuteReader`, `ExecuteNonQuery`, `ExecuteScalar`, `ExecuteXmlReader` su tipi che implementano `IDbCommand`, dove il testo del comando è costruito con stringhe non costanti (concatenazione, interpolazione con variabili, `string.Format`).
- **Falso positivo**: comandi SQL con stringhe letterali costanti.

```csharp
// 🚫 Vulnerabile
var cmd = new SqlCommand("SELECT * FROM Users WHERE Id = " + userId, conn);
cmd.ExecuteReader();

// ✅ Sicuro (parametrizzato)
var cmd = new SqlCommand("SELECT * FROM Users WHERE Id = @id", conn);
cmd.Parameters.AddWithValue("@id", userId);
cmd.ExecuteReader();
```

#### SG0002 — Command Injection

- **Cosa rileva**: chiamate a `Process.Start()` con argomenti non costanti.

```csharp
// 🚫 Vulnerabile
Process.Start("cmd.exe", "/c " + userInput);

// ✅ Sicuro
Process.Start("notepad.exe", "readme.txt");
```

#### SG0003 — Path Traversal

- **Cosa rileva**: operazioni I/O (`File.*`, `Directory.*`, `Path.*`, `FileStream`, `FileInfo`, `DirectoryInfo`) con percorsi non costanti.

```csharp
// 🚫 Vulnerabile
var content = File.ReadAllText(userInput);

// ✅ Sicuro (validazione + path combinato)
var safePath = Path.Combine(baseDir, SanitizeFileName(userInput));
var content = File.ReadAllText(safePath);
```

#### SG0004 — Insecure Deserialization

- **Cosa rileva**: creazione di `BinaryFormatter`, `NetDataContractSerializer`, `SoapFormatter`, `LosFormatter`, `ObjectStateFormatter`.

```csharp
// 🚫 Vulnerabile
var formatter = new BinaryFormatter();
var obj = formatter.Deserialize(stream);

// ✅ Sicuro
var json = JsonSerializer.Deserialize<MyType>(stream);
```

#### SG0005 — Hardcoded Secrets

- **Cosa rileva**: stringhe letterali contenenti pattern sensibili (`password`, `token`, `apikey`, `secret`, `connectionstring`) assegnate a proprietà, variabili o parametri con nomi sensibili.

```csharp
// 🚫 Vulnerabile
var connectionString = "Server=myserver;Database=mydb;User Id=sa;Password=P@ssw0rd!;";

// ✅ Sicuro
var connectionString = Configuration.GetConnectionString("MyDb");
```

#### SG0006 — Weak Cryptography

- **Cosa rileva**: chiamate a `.Create()` su `MD5`, `SHA1`, `DES`, `TripleDES`, `RC2`.
- **Code Fix**: sostituisce automaticamente con varianti sicure.

```csharp
// 🚫 Vulnerabile
var hash = MD5.Create().ComputeHash(data);

// ✅ Sicuro
var hash = SHA256.Create().ComputeHash(data);
```

#### SG0007 — Insecure Random

- **Cosa rileva**: `new Random()` (non crittografico).
- **Code Fix**: aggiunge commento TODO.

```csharp
// 🚫 Vulnerabile
var rng = new Random();

// ✅ Sicuro
var rng = RandomNumberGenerator.Create();
```

#### SG0008 — XXE (XML External Entity)

- **Cosa rileva**: `DtdProcessing.Parse` o `XmlResolver` non nullo su `XmlReaderSettings`, `XmlDocument`, `XPathDocument`.
- **Code Fix**: imposta `DtdProcessing.Prohibit` o `XmlResolver = null`.

```csharp
// 🚫 Vulnerabile
var settings = new XmlReaderSettings { DtdProcessing = DtdProcessing.Parse };

// ✅ Sicuro
var settings = new XmlReaderSettings { DtdProcessing = DtdProcessing.Prohibit };
```

#### SG0009 — TLS Validation Disabled

- **Cosa rileva**: assegnazione a `ServerCertificateValidationCallback` con lambda/delegato che restituisce sempre `true`.

```csharp
// 🚫 Vulnerabile
ServicePointManager.ServerCertificateValidationCallback = (_, _, _, _) => true;

// ✅ Sicuro (rimuovere l'override)
// ServicePointManager.ServerCertificateValidationCallback = null;
```

#### SG0010 — Sensitive Data Logging

- **Cosa rileva**: chiamate a `ILogger.Log*`, `Console.Write*`, `Trace.WriteLine`, `Debug.WriteLine` con argomenti contenenti variabili con nomi sensibili (`password`, `token`, `creditcard`, `ssn`).

```csharp
// 🚫 Vulnerabile
logger.LogInformation("User password: {Password}", userPassword);

// ✅ Sicuro
logger.LogInformation("User {UserId} logged in", user.Id);
```

#### SG0011 — LDAP Injection

- **Cosa rileva**: chiamate a `DirectorySearcher`, `DirectoryEntry` o altri metodi LDAP con filtri di ricerca costruiti da espressioni non costanti.
- **Falso positivo**: filtri LDAP con valori letterali costanti o completamente hardcoded.

```csharp
// 🚫 Vulnerabile
var searcher = new DirectorySearcher("(uid=" + userName + ")");
var result = searcher.FindOne();

// ✅ Sicuro
var searcher = new DirectorySearcher("(uid=*)");
searcher.Filter = $"(uid={EscapeLdapFilter(userName)})";
```

#### SG0012 — XPath Injection

- **Cosa rileva**: chiamate a `XPathNavigator.Select`, `XPathDocument`, `XPathExpression.Compile` o metodi `XPathEvaluate` con query costruite da espressioni non costanti.

```csharp
// 🚫 Vulnerabile
var navigator = doc.CreateNavigator();
var result = navigator.Select("//user[name='" + userName + "']");

// ✅ Sicuro
var expr = XPathExpression.Compile("//user[name=@name]");
expr.SetParameter("@name", userName);
var result = navigator.Select(expr);
```

#### SG0013 — Cross-Site Scripting (XSS)

- **Cosa rileva**: dati non attendibili scritti direttamente in una risposta HTTP senza codifica, in particolare chiamate a `Response.Write`, `Html.Raw`, o assegnazioni a `InnerHtml` con stringhe non costanti.

```csharp
// 🚫 Vulnerabile
Response.Write("<div>" + userInput + "</div>");

// ✅ Sicuro
Response.Write("<div>" + HttpUtility.HtmlEncode(userInput) + "</div>");
```

#### SG0014 — Open Redirect

- **Cosa rileva**: chiamate a `Redirect`, `RedirectPermanent`, `Response.Redirect` o `RedirectToAction` con URL non costanti (non validate contro una whitelist).

```csharp
// 🚫 Vulnerabile
return Redirect(userInput);

// ✅ Sicuro
return LocalRedirect(userInput); // Solo URL relativi
```

#### SG0015 — Server-Side Request Forgery (SSRF)

- **Cosa rileva**: chiamate a `HttpClient.GetStringAsync`, `HttpClient.GetAsync`, `HttpClient.SendAsync`, `WebClient.DownloadString` o `HttpWebRequest` con URL derivati da input utente non validato.

```csharp
// 🚫 Vulnerabile
var client = new HttpClient();
var response = await client.GetStringAsync(userInput);

// ✅ Sicuro
if (!IsAllowedDomain(new Uri(userInput))) throw new SecurityException();
var response = await client.GetStringAsync(userInput);
```

#### SG0016 — Insecure Cookie Configuration

- **Cosa rileva**: creazione di oggetti `HttpCookie` o assegnazione `CookieOptions` senza impostare `HttpOnly` e `Secure`.

```csharp
// 🚫 Vulnerabile
var cookie = new HttpCookie("session", token);
Response.AppendCookie(cookie);

// ✅ Sicuro
var cookie = new HttpCookie("session", token);
cookie.HttpOnly = true;
cookie.Secure = true;
Response.AppendCookie(cookie);
```

#### SG0017 — Weak RSA Key Size

- **Cosa rileva**: chiamate a `RSA.Create` o `new RSACryptoServiceProvider` con dimensioni di chiave inferiori a 2048 bit.

```csharp
// 🚫 Vulnerabile
var rsa = RSA.Create(1024);

// ✅ Sicuro
var rsa = RSA.Create(2048); // Minimo raccomandato
```

#### SG0018 — Missing Authorization

- **Cosa rileva**: classi controller MVC/API o action methods privi dell'attributo `[Authorize]`, che potrebbero consentire accesso non autenticato.

```csharp
// 🚫 Vulnerabile
public class AdminController : Controller { }

// ✅ Sicuro
[Authorize]
public class AdminController : Controller { }
```

#### SG0019 — Information Exposure via Error Messages

- **Cosa rileva**: accesso a proprietà `Exception.Message`, `Exception.StackTrace`, o chiamate a `Exception.ToString()` in contesti di risposta HTTP (IActionResult, ViewBag, Response.Write).

```csharp
// 🚫 Vulnerabile
catch (Exception ex)
{
    return BadRequest(ex.Message);
}

// ✅ Sicuro
catch (Exception ex)
{
    _logger.LogError(ex, "Operazione fallita");
    return BadRequest("Si è verificato un errore.");
}
```

#### SG0020 — TOCTOU Race Condition

- **Cosa rileva**: chiamate a `File.Exists` seguite da un'operazione sul file (`File.Open`, `File.ReadAllText`, `File.Delete`) senza adeguata gestione delle eccezioni, creando una finestra di race condition.

```csharp
// 🚫 Vulnerabile
if (File.Exists(path))
{
    var content = File.ReadAllText(path); // Race window
}

// ✅ Sicuro
try
{
    var content = File.ReadAllText(path);
}
catch (FileNotFoundException)
{
    // Gestisci file non trovato
}
```

---

## 6. Software Composition Analysis (SCA)

### 6.1 Funzionamento

SharpGuard SCA analizza le dipendenze NuGet del progetto e le confronta con il database **OSV.dev** (Open Source Vulnerabilities) per trovare CVE note.

**Flusso:**

```
Progetto (.csproj/.sln)
        │
        ▼
Leggi dipendenze NuGet
        │
        ▼
┌──────────────────┐
│  È in cache?     │
│  (~/.sharpguard) │
└───────┬──────────┘
        │
   ┌────┴────┐
   ▼         ▼
Cache      Non in cache
valida     o scaduta
   │         │
   │         ▼
   │  ┌──────────────┐
   │  │ OSV.dev API  │
   │  │ POST /v1/    │
   │  │ query        │
   │  └──────┬───────┘
   │         │
   │         ▼
   │    Salva in cache
   │         │
   └────┬────┘
        ▼
Risultato vulnerabilità
```

### 6.2 Cache

- **Percorso**: `~/.sharpguard/cache/`
- **Formato**: JSON, un file per ogni combinazione (nome-package, versione)
- **TTL**: 24 ore
- **Modalità offline**: con `--offline`, usa solo la cache esistente

### 6.3 Limitazioni

- Disponibile solo nel **CLI**, non nell'Analyzer NuGet (l'Analyzer non può fare I/O di rete)
- Richiede connessione a `api.osv.dev` (o cache pre-esistente)
- Rispetta il rate limiting OSV.dev con retry automatico (3 tentativi con backoff esponenziale)

---

## 7. Formati di report

SharpGuard usa il pattern **Strategy** per la generazione dei report. Attualmente supporta tre formati:

### 7.1 Console (default)

Output colorato, leggibile dall'umano, organizzato per severità:

```
=== SharpGuard Security Analysis ===
Project:   /path/to/MyApp.sln
Duration:  12.3 seconds

=== SAST Findings (3) ===

CRITICAL (1):
  SG0004 | Insecure Deserialization
         > /path/to/File.cs:42:13
         > CWE-502 (https://cwe.mitre.org/data/definitions/502.html)
         > [Code] var formatter = new BinaryFormatter();
         > [Fix]  Use JsonSerializer instead of BinaryFormatter.

=== SCA Findings (2) ===
  Package: Newtonsoft.Json 13.0.1 (source: Fresh)
    CVE-2024-XXXXX | Remote code execution in JSON parsing

=== Summary ===
  SAST: 3 findings
  SCA:  2 vulnerabilities in 1 package
  SAST by severity: 1 critical, 2 high
```

### 7.2 JSON

Output strutturato, machine-readable:

```json
{
  "tool": {
    "name": "SharpGuard",
    "version": "1.0.0"
  },
  "analysis": {
    "projectPath": "/path/to/MyApp.sln",
    "startTime": "2026-06-26T10:00:00Z",
    "duration": "00:00:12.345",
    "isOffline": false
  },
  "sast": [
    {
      "ruleId": "SG0004",
      "severity": "critical",
      "filePath": "/path/to/File.cs",
      "lineNumber": 42,
      "columnNumber": 13,
      "message": "Use JsonSerializer instead of BinaryFormatter.",
      "cwe": { "id": "502", "name": "Insecure Deserialization" }
    }
  ],
  "sca": [
    {
      "package": { "name": "Newtonsoft.Json", "version": "13.0.1" },
      "source": "fresh",
      "vulnerabilities": [
        {
          "id": "GHSA-xxxx-xxxx-xxxx",
          "aliases": ["CVE-2024-XXXXX"],
          "summary": "Remote code execution in JSON parsing"
        }
      ]
    }
  ]
}
```

### 7.3 SARIF v2.1.0

Formato standard per tool di sicurezza, compatibile con:

- **GitHub Code Scanning** (via `github/codeql-action/upload-sarif`)
- **Azure DevOps** (via `AdvancedSecurity-CodeScan`)
- **Visual Studio** (apertura diretta di file `.sarif`)

Mappatura severità:

| SharpGuard | SARIF level |
|------------|-------------|
| Critical   | `error` |
| High       | `error` |
| Medium     | `warning` |
| Low        | `note` |
| Info       | `none` |

---

## 8. Configurazione severità (.editorconfig)

La severità di ogni regola si configura tramite file `.editorconfig`:

```ini
# .editorconfig
[*.cs]

# SQL injection come errore bloccante
dotnet_diagnostic.SG0001.severity = error

# Disabilita Sensitive Data Logging nei test
dotnet_diagnostic.SG0010.severity = none

# Path Traversal come warning
dotnet_diagnostic.SG0003.severity = warning
```

**Valori supportati**: `error`, `warning`, `info`, `hidden`, `none` (disabilita).

---

## 9. Integrazione CI/CD

### 9.1 GitHub Actions

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

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Install SharpGuard
        run: dotnet tool install --global SharpGuard

      - name: Run SAST + SCA
        run: |
          SharpGuard analyze \
            --path ./src \
            --format sarif \
            --severity-threshold high \
            --output sharpguard.sarif \
            --fail-on-findings

      - name: Upload SARIF to GitHub Code Scanning
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: sharpguard.sarif
```

### 9.2 Azure DevOps

```yaml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

steps:
  - task: DotNetCoreCLI@2
    displayName: 'Install SharpGuard'
    inputs:
      command: custom
      custom: tool
      arguments: 'install --global SharpGuard'

  - script: |
      SharpGuard analyze \
        --path $(Build.SourcesDirectory) \
        --format sarif \
        --output $(Build.ArtifactStagingDirectory)/sharpguard.sarif \
        --fail-on-findings
    displayName: 'Run SharpGuard Security Analysis'

  - task: PublishBuildArtifacts@1
    inputs:
      pathToPublish: '$(Build.ArtifactStagingDirectory)/sharpguard.sarif'
      artifactName: 'sharpguard-report'
```

### 9.3 GitLab CI

```yaml
sharpguard:
  stage: test
  image: mcr.microsoft.com/dotnet/sdk:8.0
  script:
    - dotnet tool install --global SharpGuard
    - SharpGuard analyze --path ./src --format sarif --output sharpguard.sarif --fail-on-findings
  artifacts:
    paths:
      - sharpguard.sarif
```

---

## Licenza

MIT — vedi [LICENSE](../LICENSE).

## Autore

Paolino Salmone — [https://lupala.com](https://lupala.com)
