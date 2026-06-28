# SharpGuard Claude Agent

Agente AI per [Claude Code](https://claude.ai/code) che rileva e corregge automaticamente vulnerabilità di sicurezza in codebase C# usando SharpGuard.

## Cosa fa

- Esegue scansioni SAST + SCA con SharpGuard
- Classifica i finding per severità e tipo
- Applica fix automatiche per le 10 regole OWASP
- Verifica build e test dopo ogni modifica
- Itera fino a zero finding (o minimo raggiungibile)

## Installazione

### 1. Installa SharpGuard

```bash
dotnet tool install --global SharpGuard
```

### 2. Installa l'agente Claude

Copia il file `SharpGuard.agent.md` nella directory degli agenti globali di Claude Code:

```bash
mkdir -p ~/.claude/agents
cp SharpGuard.agent.md ~/.claude/agents/
```

Oppure in una shell:

```bash
curl -o ~/.claude/agents/SharpGuard.agent.md \
  https://raw.githubusercontent.com/Lupala-coder/SharpGuard/main/claude-agent/SharpGuard.agent.md
```

### 3. Verifica

Riavvia Claude Code. SharpGuard apparirà tra gli agent disponibili.

## Utilizzo

In Claude Code, invoca l'agente quando vuoi analizzare e correggere vulnerabilità in un progetto .NET:

> *"Usa l'agente SharpGuard per analizzare questa solution e correggere tutte le vulnerabilità"*

L'agente:
1. Rileva automaticamente la solution `.sln`
2. Esegue `SharpGuard analyze` con output JSON
3. Per ogni finding, applica la strategia di fix specifica per quella regola
4. Compila e testa dopo ogni modifica
5. Itera fino a esaurimento dei finding
6. Produce un report finale con il riepilogo

## Regole coperte

| ID | Vulnerabilità | Severità |
|----|---------------|----------|
| SG0001 | SQL Injection | High |
| SG0002 | Command Injection | High |
| SG0003 | Path Traversal | Medium |
| SG0004 | Insecure Deserialization | Critical |
| SG0005 | Hardcoded Secrets | Critical |
| SG0006 | Weak Cryptography | High |
| SG0007 | Insecure Random | Medium |
| SG0008 | XXE | High |
| SG0009 | TLS Validation Disabled | High |
| SG0010 | Sensitive Data Logging | Medium |

## Vincoli di sicurezza

L'agente **non farà mai**:
- Eliminare o rinominare file senza conferma
- Modificare struttura del progetto
- Alterare test per farli passare
- Sopprimere warning con `#pragma`
- Committare automaticamente
- Eseguire comandi distruttivi

## Requisiti

- Claude Code (qualsiasi versione)
- .NET 8+ SDK
- SharpGuard installato come global tool

## Licenza

MIT — vedi [LICENSE](../LICENSE).
