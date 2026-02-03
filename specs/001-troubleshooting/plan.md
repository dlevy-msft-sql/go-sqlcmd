# Implementation Plan: SQL Server Connection Troubleshooting

**Branch**: `001-troubleshooting` | **Date**: 2026-02-02 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-troubleshooting/spec.md`

## Summary

Add comprehensive troubleshooting capabilities to go-sqlcmd, enabling users to diagnose common SQL Server connection issues through automated analysis of connectivity, Kerberos/SSPI configuration, TLS certificates, and logs. The implementation follows the existing go-sqlcmd patterns using Cobra for the CLI and platform abstraction for cross-platform support.

## Technical Context

**Language/Version**: Go 1.21+  
**Primary Dependencies**: go-mssqldb, Cobra, existing go-sqlcmd packages  
**Storage**: N/A (diagnostic tool)  
**Testing**: Go testing, testify/assert  
**Target Platform**: Windows, Linux, macOS  
**Project Type**: Single CLI application (existing)  
**Performance Goals**: Basic connectivity test < 30 seconds, full report < 2 minutes

**Constraints**: 
- Must work without elevated privileges for basic tests
- Must gracefully degrade when platform features unavailable
- Must follow existing go-sqlcmd localization patterns

**Scale/Scope**: Single-user CLI tool, diagnostics run on demand

## Constitution Check

*Based on go-sqlcmd project conventions from `.github/copilot-instructions.md`*

| Principle | Status | Notes |
|-----------|--------|-------|
| Follow existing CLI patterns | ✅ | Use Cobra, follow cmd/modern/root/ structure |
| Use localizer for messages | ✅ | All user-facing strings via internal/localizer |
| Platform abstraction | ✅ | Use internal/pal pattern for OS-specific code |
| Test coverage | ✅ | Unit tests with testify, mock AD/registry access |
| Error handling | ✅ | Use localizer.Errorf, wrap with context |

## Project Structure

### Documentation (this feature)

```text
specs/001-troubleshooting/
├── spec.md              # Feature specification
├── research.md          # Domain knowledge from CSS team
├── data-model.md        # Go struct definitions
├── quickstart.md        # Usage guide
├── plan.md              # This file
└── tasks.md             # Task breakdown (next step)
```

### Source Code (repository root)

```text
cmd/modern/root/
├── troubleshoot.go           # Main troubleshoot command
└── troubleshoot/
    ├── connectivity.go       # Connectivity subcommand
    ├── kerberos.go           # Kerberos subcommand
    ├── certificate.go        # Certificate subcommand
    ├── logs.go               # Logs subcommand
    └── report.go             # Report generation

internal/
├── troubleshoot/
│   ├── doc.go                # Package documentation
│   ├── types.go              # Data model types
│   ├── config.go             # Configuration handling
│   ├── result.go             # Result aggregation
│   └── remediation.go        # Remediation generation
│
├── troubleshoot/connectivity/
│   ├── connectivity.go       # Connectivity checks
│   ├── dns.go                # DNS resolution
│   ├── tcp.go                # TCP port testing
│   ├── sqlbrowser.go         # SQL Browser protocol
│   └── namedpipes_windows.go # Named pipes (Windows)
│
├── troubleshoot/kerberos/
│   ├── kerberos.go           # Kerberos orchestration
│   ├── spn.go                # SPN validation
│   ├── encryption.go         # Encryption type checks
│   ├── ticket.go             # Ticket cache analysis
│   ├── ad_windows.go         # AD queries (Windows)
│   ├── ad_linux.go           # AD queries (Linux/SSSD)
│   └── krb5_unix.go          # krb5.conf parsing (Unix)
│
├── troubleshoot/certificate/
│   ├── certificate.go        # Certificate orchestration
│   ├── validation.go         # SAN/expiration/chain validation
│   ├── store_windows.go      # Windows certificate store
│   ├── store_linux.go        # Linux certificate handling
│   └── store_darwin.go       # macOS Keychain
│
├── troubleshoot/logs/
│   ├── logs.go               # Log analysis orchestration
│   ├── parser.go             # Log parsing utilities
│   ├── errorlog.go           # SQL Server ERRORLOG
│   ├── eventlog_windows.go   # Windows Event Log
│   ├── journald_linux.go     # Linux journald
│   └── syslog_darwin.go      # macOS logs
│
└── troubleshoot/output/
    ├── output.go             # Output formatting
    ├── text.go               # Text formatter
    ├── json.go               # JSON formatter
    └── html.go               # HTML report generator

pkg/sqlcmd/
└── troubleshoot.go           # Public API for troubleshooting (optional)
```

## Implementation Phases

### Phase 0: Foundation
- Create package structure
- Define types from data-model.md
- Set up platform abstraction patterns

### Phase 1: Basic Connectivity (User Story 1)
- DNS resolution
- TCP port testing
- SQL Browser protocol
- Named pipes (Windows)
- CLI integration

### Phase 2: Connection String Validation (User Story 2)
- Connection string parsing
- Component validation
- Error categorization
- CLI integration

### Phase 3: Kerberos Diagnostics (User Story 3)
- SPN queries and validation
- Encryption type analysis
- Ticket cache inspection
- Remediation generation
- Platform-specific implementations

### Phase 4: Certificate Validation (User Story 4)
- Certificate store access
- SAN validation
- Expiration/chain checks
- Permission verification

### Phase 5: Log Analysis (User Story 5)
- SQL Server ERRORLOG parsing
- Windows Event Log reading
- Linux/macOS log integration
- Error categorization

### Phase 6: Reporting (User Story 6)
- Text output
- JSON output
- HTML report generation
- Redaction support

## Dependencies & Risks

### Dependencies
- Windows AD queries require domain-joined machine or RSAT
- Certificate store access requires appropriate permissions
- Log access may require elevated privileges

### Risks
| Risk | Mitigation |
|------|------------|
| AD query failures | Graceful degradation with clear messaging |
| Platform differences in Kerberos | Abstract behind interface, test per-platform |
| Log format variations | Flexible parsing with multiple patterns |
| Performance with large logs | Time-based filtering, streaming processing |

## Testing Strategy

### Unit Tests
- Mock AD/LDAP responses
- Mock certificate store
- Mock log file contents
- Test each validator independently

### Integration Tests  
- Require test SQL Server instance
- Test connectivity scenarios
- Test with actual certificates

### Platform Tests
- Windows: Full AD integration tests
- Linux: krb5.conf parsing tests
- macOS: Basic functionality tests

## Localization

All user-facing strings must use `internal/localizer`:

```go
// Example
localizer.Sprintf("Checking connectivity to %s...", serverName)
localizer.Errorf("Cannot resolve hostname: %s", hostname)
```

## Open Items for Implementation

1. Decide on public API in `pkg/sqlcmd/troubleshoot.go` vs internal only
2. Confirm HTML report template requirements
3. Determine if network trace analysis is in scope
4. Azure SQL Database support scope
