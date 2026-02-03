# Data Model: SQL Server Connection Troubleshooting

**Feature**: 001-troubleshooting  
**Date**: 2026-02-02

## Overview

This document defines the data structures used for the troubleshooting feature. All entities are designed to be serializable to JSON for output and automation purposes.

---

## Core Entities

### TroubleshootConfig

Configuration for a troubleshooting session.

```go
type TroubleshootConfig struct {
    // Target server information
    ServerName      string        `json:"serverName"`
    Port            int           `json:"port,omitempty"`
    InstanceName    string        `json:"instanceName,omitempty"`
    
    // Connection string (alternative to server/port)
    ConnectionString string       `json:"connectionString,omitempty"`
    
    // What to check
    CheckConnectivity  bool       `json:"checkConnectivity"`
    CheckKerberos      bool       `json:"checkKerberos"`
    CheckCertificate   bool       `json:"checkCertificate"`
    CheckLogs          bool       `json:"checkLogs"`
    
    // Filters
    LogTimeRangeStart  time.Time  `json:"logTimeRangeStart,omitempty"`
    LogTimeRangeEnd    time.Time  `json:"logTimeRangeEnd,omitempty"`
    
    // Output options
    OutputFormat    OutputFormat  `json:"outputFormat"`
    RedactSensitive bool          `json:"redactSensitive"`
    Timeout         time.Duration `json:"timeout"`
}

type OutputFormat string

const (
    OutputFormatText OutputFormat = "text"
    OutputFormatJSON OutputFormat = "json"
    OutputFormatHTML OutputFormat = "html"
)
```

---

### TroubleshootResult

Top-level result containing all diagnostic information.

```go
type TroubleshootResult struct {
    // Metadata
    Timestamp    time.Time     `json:"timestamp"`
    Duration     time.Duration `json:"duration"`
    Platform     string        `json:"platform"`
    ToolVersion  string        `json:"toolVersion"`
    
    // Target
    Config       TroubleshootConfig `json:"config"`
    
    // Results
    OverallStatus   Status              `json:"overallStatus"`
    Connectivity    *ConnectivityResult `json:"connectivity,omitempty"`
    Kerberos        *KerberosResult     `json:"kerberos,omitempty"`
    Certificate     *CertificateResult  `json:"certificate,omitempty"`
    Logs            *LogAnalysisResult  `json:"logs,omitempty"`
    
    // Collected remediations from all checks
    Remediations []Remediation `json:"remediations,omitempty"`
    
    // Errors encountered during analysis
    Errors []DiagnosticError `json:"errors,omitempty"`
}

type Status string

const (
    StatusPass     Status = "pass"
    StatusFail     Status = "fail"
    StatusWarning  Status = "warning"
    StatusSkipped  Status = "skipped"
    StatusUnknown  Status = "unknown"
)
```

---

## Connectivity Entities

### ConnectivityResult

```go
type ConnectivityResult struct {
    Status       Status                `json:"status"`
    
    // DNS resolution
    DNS          *DNSResult            `json:"dns,omitempty"`
    
    // Port connectivity
    TCPPort      *TCPPortResult        `json:"tcpPort,omitempty"`
    
    // SQL Browser (for named instances)
    SQLBrowser   *SQLBrowserResult     `json:"sqlBrowser,omitempty"`
    
    // Named pipes (Windows only)
    NamedPipes   *NamedPipesResult     `json:"namedPipes,omitempty"`
    
    // Latency measurement
    LatencyMs    float64               `json:"latencyMs,omitempty"`
}

type DNSResult struct {
    Status      Status   `json:"status"`
    Hostname    string   `json:"hostname"`
    ResolvedIPs []string `json:"resolvedIPs,omitempty"`
    Error       string   `json:"error,omitempty"`
}

type TCPPortResult struct {
    Status     Status `json:"status"`
    Port       int    `json:"port"`
    IsOpen     bool   `json:"isOpen"`
    Error      string `json:"error,omitempty"`
    // Suggestions if blocked
    Suggestions []string `json:"suggestions,omitempty"`
}

type SQLBrowserResult struct {
    Status       Status `json:"status"`
    IsResponding bool   `json:"isResponding"`
    InstancePort int    `json:"instancePort,omitempty"`
    Instances    []SQLInstance `json:"instances,omitempty"`
    Error        string `json:"error,omitempty"`
}

type SQLInstance struct {
    Name     string `json:"name"`
    Port     int    `json:"port,omitempty"`
    IsClustered bool `json:"isClustered,omitempty"`
    Version  string `json:"version,omitempty"`
}

type NamedPipesResult struct {
    Status   Status `json:"status"`
    PipeName string `json:"pipeName"`
    IsOpen   bool   `json:"isOpen"`
    Error    string `json:"error,omitempty"`
}
```

---

## Kerberos Entities

### KerberosResult

```go
type KerberosResult struct {
    Status           Status                   `json:"status"`
    
    // Service Principal Names
    SPNCheck         *SPNCheckResult          `json:"spnCheck,omitempty"`
    
    // Encryption types
    EncryptionCheck  *EncryptionCheckResult   `json:"encryptionCheck,omitempty"`
    
    // Ticket cache
    TicketCache      *TicketCacheResult       `json:"ticketCache,omitempty"`
    
    // Summary of issues
    Issues           []KerberosIssue          `json:"issues,omitempty"`
}

type SPNCheckResult struct {
    Status            Status        `json:"status"`
    ExpectedSPNs      []string      `json:"expectedSpns"`
    FoundSPNs         []SPNInfo     `json:"foundSpns,omitempty"`
    DuplicateSPNs     []SPNDuplicate `json:"duplicateSpns,omitempty"`
    MissingSPNs       []string      `json:"missingSpns,omitempty"`
    WrongAccountSPNs  []SPNInfo     `json:"wrongAccountSpns,omitempty"`
}

type SPNInfo struct {
    SPN          string `json:"spn"`
    Account      string `json:"account"`
    AccountType  string `json:"accountType"` // "user", "computer", "gmsa"
    IsCorrect    bool   `json:"isCorrect"`
}

type SPNDuplicate struct {
    SPN      string   `json:"spn"`
    Accounts []string `json:"accounts"`
}

type EncryptionCheckResult struct {
    Status                 Status              `json:"status"`
    ServiceAccount         *EncryptionTypeInfo `json:"serviceAccount,omitempty"`
    ComputerAccount        *EncryptionTypeInfo `json:"computerAccount,omitempty"`
    LocalPolicy            *EncryptionTypeInfo `json:"localPolicy,omitempty"`
    Mismatch               bool                `json:"mismatch"`
    MismatchDescription    string              `json:"mismatchDescription,omitempty"`
}

type EncryptionTypeInfo struct {
    AccountName      string   `json:"accountName"`
    RawValue         int      `json:"rawValue"`
    SupportedTypes   []string `json:"supportedTypes"` // ["RC4", "AES128", "AES256"]
    Source           string   `json:"source"` // "AD", "LocalPolicy", "krb5.conf"
}

type TicketCacheResult struct {
    Status        Status       `json:"status"`
    HasTicket     bool         `json:"hasTicket"`
    TicketInfo    *TicketInfo  `json:"ticketInfo,omitempty"`
}

type TicketInfo struct {
    ServicePrincipal string    `json:"servicePrincipal"`
    EncryptionType   string    `json:"encryptionType"`
    StartTime        time.Time `json:"startTime"`
    EndTime          time.Time `json:"endTime"`
    RenewTill        time.Time `json:"renewTill,omitempty"`
}

type KerberosIssue struct {
    Severity    Severity    `json:"severity"`
    Category    string      `json:"category"` // "encryption", "spn", "ticket"
    Description string      `json:"description"`
    Remediation *Remediation `json:"remediation,omitempty"`
}

type Severity string

const (
    SeverityError   Severity = "error"
    SeverityWarning Severity = "warning"
    SeverityInfo    Severity = "info"
)
```

---

## Certificate Entities

### CertificateResult

```go
type CertificateResult struct {
    Status             Status                  `json:"status"`
    
    // SQL Server certificate configuration
    ForceEncryption    bool                    `json:"forceEncryption"`
    ConfiguredThumbprint string                `json:"configuredThumbprint,omitempty"`
    
    // Certificate details
    CertificateInfo    *CertificateInfo        `json:"certificateInfo,omitempty"`
    
    // Validation results
    SANValidation      *SANValidationResult    `json:"sanValidation,omitempty"`
    ExpirationCheck    *ExpirationCheckResult  `json:"expirationCheck,omitempty"`
    PermissionCheck    *PermissionCheckResult  `json:"permissionCheck,omitempty"`
    ChainValidation    *ChainValidationResult  `json:"chainValidation,omitempty"`
    
    // Issues found
    Issues             []CertificateIssue      `json:"issues,omitempty"`
}

type CertificateInfo struct {
    Thumbprint       string    `json:"thumbprint"`
    Subject          string    `json:"subject"`
    Issuer           string    `json:"issuer"`
    NotBefore        time.Time `json:"notBefore"`
    NotAfter         time.Time `json:"notAfter"`
    SANEntries       []string  `json:"sanEntries,omitempty"`
    HasPrivateKey    bool      `json:"hasPrivateKey"`
    KeyUsage         []string  `json:"keyUsage,omitempty"`
    EKU              []string  `json:"eku,omitempty"` // Extended Key Usage
}

type SANValidationResult struct {
    Status           Status   `json:"status"`
    ConnectionName   string   `json:"connectionName"`
    MatchingSAN      string   `json:"matchingSan,omitempty"`
    IsMatch          bool     `json:"isMatch"`
    CheckedSANs      []string `json:"checkedSans"`
    Suggestion       string   `json:"suggestion,omitempty"`
}

type ExpirationCheckResult struct {
    Status        Status    `json:"status"`
    NotAfter      time.Time `json:"notAfter"`
    DaysRemaining int       `json:"daysRemaining"`
    IsExpired     bool      `json:"isExpired"`
    IsExpiringSoon bool     `json:"isExpiringSoon"` // < 30 days
}

type PermissionCheckResult struct {
    Status             Status `json:"status"`
    ServiceAccount     string `json:"serviceAccount"`
    HasReadPermission  bool   `json:"hasReadPermission"`
    CurrentPermissions []string `json:"currentPermissions,omitempty"`
}

type ChainValidationResult struct {
    Status       Status          `json:"status"`
    IsValid      bool            `json:"isValid"`
    ChainErrors  []string        `json:"chainErrors,omitempty"`
    Chain        []ChainCertInfo `json:"chain,omitempty"`
}

type ChainCertInfo struct {
    Subject    string `json:"subject"`
    Issuer     string `json:"issuer"`
    Thumbprint string `json:"thumbprint"`
}

type CertificateIssue struct {
    Severity    Severity    `json:"severity"`
    Category    string      `json:"category"` // "san", "expiration", "permission", "chain"
    Description string      `json:"description"`
    Remediation *Remediation `json:"remediation,omitempty"`
}
```

---

## Log Analysis Entities

### LogAnalysisResult

```go
type LogAnalysisResult struct {
    Status        Status               `json:"status"`
    TimeRange     TimeRange            `json:"timeRange"`
    
    // Categorized entries
    Errors        []LogEntry           `json:"errors,omitempty"`
    Warnings      []LogEntry           `json:"warnings,omitempty"`
    
    // Summary by category
    Summary       []LogCategorySummary `json:"summary,omitempty"`
    
    // Sources analyzed
    Sources       []LogSource          `json:"sources"`
}

type TimeRange struct {
    Start time.Time `json:"start"`
    End   time.Time `json:"end"`
}

type LogEntry struct {
    Timestamp   time.Time `json:"timestamp"`
    Source      string    `json:"source"` // "ERRORLOG", "EventLog-Application", etc.
    Severity    Severity  `json:"severity"`
    Category    string    `json:"category"` // "authentication", "connectivity", "tls", etc.
    Message     string    `json:"message"`
    EventID     int       `json:"eventId,omitempty"` // Windows Event ID
    ProcessID   int       `json:"processId,omitempty"`
    
    // Related information
    UserName    string    `json:"userName,omitempty"`
    ClientIP    string    `json:"clientIp,omitempty"`
    DatabaseName string   `json:"databaseName,omitempty"`
}

type LogCategorySummary struct {
    Category    string `json:"category"`
    ErrorCount  int    `json:"errorCount"`
    WarningCount int   `json:"warningCount"`
    FirstSeen   time.Time `json:"firstSeen"`
    LastSeen    time.Time `json:"lastSeen"`
}

type LogSource struct {
    Name        string `json:"name"`
    Path        string `json:"path,omitempty"`
    Status      Status `json:"status"`
    EntriesRead int    `json:"entriesRead"`
    Error       string `json:"error,omitempty"`
}
```

---

## Remediation Entities

### Remediation

Represents a suggested fix for an identified issue.

```go
type Remediation struct {
    ID              string           `json:"id"`
    Title           string           `json:"title"`
    Description     string           `json:"description"`
    Category        string           `json:"category"` // "kerberos", "certificate", "connectivity"
    Priority        int              `json:"priority"` // 1 = highest
    
    // Commands to fix the issue
    Commands        []RemediationCommand `json:"commands,omitempty"`
    
    // Documentation
    DocumentationURL string          `json:"documentationUrl,omitempty"`
    
    // Impact information
    RequiresRestart  bool            `json:"requiresRestart,omitempty"`
    RequiresADAdmin  bool            `json:"requiresAdAdmin,omitempty"`
    RequiresLocalAdmin bool          `json:"requiresLocalAdmin,omitempty"`
}

type RemediationCommand struct {
    Description  string   `json:"description"`
    Platform     string   `json:"platform"` // "windows", "linux", "any"
    Shell        string   `json:"shell"`    // "powershell", "bash", "cmd"
    Command      string   `json:"command"`
    Caution      string   `json:"caution,omitempty"` // Warnings before running
}
```

---

## Error Entities

### DiagnosticError

Represents an error encountered during diagnostic checks.

```go
type DiagnosticError struct {
    Check       string `json:"check"`       // Which check failed
    Code        string `json:"code"`        // Error code for programmatic handling
    Message     string `json:"message"`
    Suggestion  string `json:"suggestion,omitempty"`
    CanContinue bool   `json:"canContinue"` // Whether other checks can proceed
}
```

---

## Enums Summary

```go
// Status indicates the result of a check
type Status string
const (
    StatusPass     Status = "pass"
    StatusFail     Status = "fail"
    StatusWarning  Status = "warning"
    StatusSkipped  Status = "skipped"
    StatusUnknown  Status = "unknown"
)

// Severity indicates the severity of an issue
type Severity string
const (
    SeverityError   Severity = "error"
    SeverityWarning Severity = "warning"
    SeverityInfo    Severity = "info"
)

// OutputFormat for report generation
type OutputFormat string
const (
    OutputFormatText OutputFormat = "text"
    OutputFormatJSON OutputFormat = "json"
    OutputFormatHTML OutputFormat = "html"
)
```

---

## JSON Examples

### Sample TroubleshootResult (Kerberos Issue)

```json
{
  "timestamp": "2026-02-02T14:30:00Z",
  "duration": "5.2s",
  "platform": "windows",
  "toolVersion": "1.5.0",
  "overallStatus": "fail",
  "kerberos": {
    "status": "fail",
    "encryptionCheck": {
      "status": "fail",
      "serviceAccount": {
        "accountName": "DOMAIN\\sqlsvc",
        "rawValue": 4,
        "supportedTypes": ["RC4"],
        "source": "AD"
      },
      "computerAccount": {
        "accountName": "DOMAIN\\SQLSERVER01$",
        "rawValue": 24,
        "supportedTypes": ["AES128", "AES256"],
        "source": "AD"
      },
      "mismatch": true,
      "mismatchDescription": "Service account only supports RC4, but computer accounts require AES"
    },
    "issues": [
      {
        "severity": "error",
        "category": "encryption",
        "description": "Encryption type mismatch: service account DOMAIN\\sqlsvc only supports RC4, but clients require AES",
        "remediation": {
          "id": "KERB-001",
          "title": "Update Service Account Encryption Types",
          "commands": [
            {
              "description": "Add AES support to service account",
              "platform": "windows",
              "shell": "powershell",
              "command": "Set-ADUser -Identity 'sqlsvc' -Replace @{'msDS-SupportedEncryptionTypes'=28}"
            }
          ],
          "requiresAdAdmin": true
        }
      }
    ]
  },
  "remediations": [
    {
      "id": "KERB-001",
      "title": "Update Service Account Encryption Types",
      "description": "The SQL Server service account only supports RC4 encryption, but client machines or policies require AES.",
      "category": "kerberos",
      "priority": 1,
      "commands": [
        {
          "description": "Add AES support to service account",
          "platform": "windows",
          "shell": "powershell",
          "command": "Set-ADUser -Identity 'sqlsvc' -Replace @{'msDS-SupportedEncryptionTypes'=28}",
          "caution": "This change affects Kerberos authentication. Ensure no active connections before applying."
        }
      ],
      "documentationUrl": "https://learn.microsoft.com/sql/database-engine/configure-windows/register-a-service-principal-name-for-kerberos-connections",
      "requiresAdAdmin": true
    }
  ]
}
```
