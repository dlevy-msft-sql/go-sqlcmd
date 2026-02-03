# Quickstart: SQL Server Connection Troubleshooting

**Feature**: 001-troubleshooting

## Prerequisites

- go-sqlcmd installed (`sqlcmd` available in PATH)
- Network access to the SQL Server you want to troubleshoot
- For Kerberos diagnostics on Windows: Domain-joined machine or RSAT tools installed
- For network tracing: Administrator/root privileges
- For certificate diagnostics: Access to the SQL Server machine (local or remote admin)

## Quick Commands

### Collect System Information (like SQL Check)

```bash
# Get comprehensive system info
sqlcmd troubleshoot --sysinfo

# Show current user context
sqlcmd troubleshoot --sysinfo --who-am-i

# List all ODBC drivers
sqlcmd troubleshoot --sysinfo --drivers

# Check TLS/cipher configuration
sqlcmd troubleshoot --sysinfo --tls
```

### Test Basic Connectivity

```bash
# Test if SQL Server is reachable
sqlcmd troubleshoot --server myserver.domain.com

# Test specific port
sqlcmd troubleshoot --server myserver.domain.com --port 1433

# Test named instance (queries SQL Browser)
sqlcmd troubleshoot --server myserver.domain.com --instance MYINSTANCE

# Flush DNS and Kerberos before testing (recommended)
sqlcmd troubleshoot --server myserver.domain.com --flush
```

### Validate Connection String

```bash
# Test your application's connection string
sqlcmd troubleshoot --connection-string "Server=myserver;Database=mydb;Integrated Security=true;Encrypt=true"

# With specific driver context
sqlcmd troubleshoot --connection-string "Server=myserver;..." --driver go-mssqldb
```

### Connection Timing Analysis

```bash
# See where time is spent during connection
sqlcmd troubleshoot --server myserver.domain.com --timing

# Output shows:
#   DNS Resolution:     12ms
#   TCP Socket Open:    45ms  
#   SSL/TLS Handshake: 180ms
#   Login Phase:       2340ms  ← BOTTLENECK (SQL Server perf issue)
```

### Collect Driver Diagnostic Trace (BID Trace)

```bash
# Enable tracing, connect, collect trace - all in one command
sqlcmd troubleshoot --server myserver.domain.com --trace

# Specify output location
sqlcmd troubleshoot --server myserver.domain.com --trace --trace-output ./mytrace.etl
```

### Collect Network Trace

```bash
# Capture network packets during connection (requires admin)
sqlcmd troubleshoot --server myserver.domain.com --nettrace

# Combine driver trace and network trace
sqlcmd troubleshoot --server myserver.domain.com --trace --nettrace

# Filter to specific port only
sqlcmd troubleshoot --server myserver.domain.com --nettrace --port 1433
```

### Retry/Stress Testing (Intermittent Issues)

```bash
# Try 100 connections with 500ms delay between each
sqlcmd troubleshoot --server myserver.domain.com --retry 100 --delay 500

# Stop on first failure
sqlcmd troubleshoot --server myserver.domain.com --retry 100 --delay 500 --stop-on-failure

# Collect traces during retry test
sqlcmd troubleshoot --server myserver.domain.com --retry 100 --delay 500 --trace
```

### Diagnose Kerberos Issues

```bash
# Check Kerberos configuration for a server
sqlcmd troubleshoot --server myserver.domain.com --kerberos

# Includes: SPN validation, encryption type checks, ticket cache analysis
```

### Validate TLS/Certificate

```bash
# Check certificate configuration
sqlcmd troubleshoot --server myserver.domain.com --certificate

# Validates: SAN entries, expiration, permissions, chain, cipher issues
```

### Analyze Logs

```bash
# Scan logs for connection errors
sqlcmd troubleshoot --server myserver.domain.com --logs

# Filter by time range
sqlcmd troubleshoot --server myserver.domain.com --logs --since "2026-02-01" --until "2026-02-02"
```

### Generate Full Report Package

```bash
# Collect everything and package into ZIP
sqlcmd troubleshoot --server myserver.domain.com --full-report --output diagnostic.zip

# Generate JSON for automation
sqlcmd troubleshoot --server myserver.domain.com --full-report --output-format json

# Redact sensitive info for sharing
sqlcmd troubleshoot --server myserver.domain.com --full-report --output report.zip --redact
```

## Common Scenarios

### "Cannot Generate SSPI Context" Error

```bash
# Run Kerberos diagnostics
sqlcmd troubleshoot --server myserver.domain.com --kerberos

# The tool will check:
# 1. Service account encryption types (RC4 vs AES)
# 2. SPN registration
# 3. Duplicate SPNs
# 4. Local policy restrictions
# 5. DNS suffix in TGS request
#
# And provide PowerShell commands to fix any issues found
```

### Connection Timeout

```bash
# Check connectivity step by step with timing
sqlcmd troubleshoot --server myserver.domain.com --connectivity --timing

# This will show:
# - DNS resolution time
# - TCP port reachability and latency
# - SSL/TLS handshake time
# - Login phase time (if slow here, it's SQL Server, not client!)
# - SQL Browser response (for named instances)
```

### Intermittent Connection Failures

```bash
# Run multiple connection attempts with tracing
sqlcmd troubleshoot --server myserver.domain.com --retry 500 --delay 1000 --trace --nettrace

# This collects:
# - 500 connection attempts with 1 second delay
# - Driver diagnostic trace for each
# - Network capture showing exactly what happened on the wire
# - Report of which attempts failed and why
```

### Certificate/TLS Errors

```bash
# Validate certificate for encrypted connections
sqlcmd troubleshoot --server myserver.domain.com --certificate --connection-name "mylistener.domain.com"

# Check if connection name is in certificate SAN
# Verify certificate hasn't expired
# Check service account permissions on private key
# Detect known cipher issues (Diffie-Hellman bug)
```

### Filter Driver Blocking Connections

If you suspect a filter driver (antivirus, firewall software):

```bash
# Collect both traces
sqlcmd troubleshoot --server myserver.domain.com --trace --nettrace

# If BID trace shows TCP open was called but network trace shows
# no packets were sent, something in between is blocking - likely
# a filter driver like CrowdStrike, etc.
```

### Linux ODBC Configuration Issues

```bash
# Dump all ODBC INI settings and DSN parameters
sqlcmd troubleshoot --sysinfo --odbc

# Test connection with all parameters visible
sqlcmd troubleshoot --connection-string "DSN=mydsn" --verbose
```

### Kerberos Delegation (Double-Hop) Issues

```bash
# Check delegation settings for service account
sqlcmd troubleshoot --server myserver.domain.com --kerberos --check-delegation

# Shows:
# - Whether delegation is enabled
# - Type: unconstrained, constrained, or resource-based
# - Which services the account can delegate to
```

### Production-Safe Testing (Listener Mode - Stretch Goal)

Test network connectivity without touching SQL Server:

```bash
# On the target machine (where SQL Server runs)
sqlcmd listen --port 14330

# On the client machine
sqlcmd troubleshoot --listener-test --server targetmachine --port 14330

# This proves network works without involving SQL Server
# Safe to run during production peak times
```

### Two-Side Testing (Find Network Break Point)

```bash
# On machine A
sqlcmd listen --port 14330 --also-test-to machineB:14330

# On machine B  
sqlcmd listen --port 14330 --also-test-to machineA:14330

# Both sides try to connect to each other
# Report shows where in the network path connectivity breaks
```

## Output Formats

| Format | Flag | Use Case |
|--------|------|----------|
| Text | `--output-format text` (default) | Interactive troubleshooting |
| JSON | `--output-format json` | Automation, scripting |
| ZIP Package | `--output file.zip` | Complete data collection for CSS |

## Files in Report Package

When using `--full-report --output diagnostic.zip`:

```
diagnostic.zip
├── sysinfo.json          # System information
├── connectivity.json     # Connection test results
├── timing.json           # Connection phase timings
├── kerberos.json         # Kerberos analysis (if applicable)
├── certificate.json      # Certificate validation (if applicable)
├── driver_trace.etl      # BID trace (if --trace used)
├── network_trace.etl     # Network capture (if --nettrace used)
├── logs/
│   ├── errorlog.txt      # SQL Server error log excerpts
│   └── events.txt        # Windows Event Log excerpts
└── README.txt            # Upload instructions
```

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | All checks passed |
| 1 | One or more checks failed |
| 2 | Configuration/usage error |
| 3 | Network/connectivity error |
| 4 | Permission denied (need admin) |

## Platform Notes

### Windows
- Full Kerberos diagnostics with AD integration
- Certificate validation via Windows Certificate Store
- Named Pipes connectivity testing
- Windows Event Log analysis

### Linux
- Kerberos via krb5.conf and keytab analysis
- Certificate validation if path provided
- journalctl/syslog analysis
- AD integration if domain-joined via SSSD

### macOS
- Similar to Linux for Kerberos
- Console/system.log analysis
- Keychain certificate access

## Next Steps

After identifying issues, the tool provides:
1. **Clear explanation** of what's wrong
2. **PowerShell/CLI commands** to fix the issue
3. **Documentation links** for more information

Example output:
```
❌ KERBEROS: Encryption type mismatch detected

   Service account: DOMAIN\sqlsvc
   Supported types: RC4 only
   
   Client policy requires: AES256, AES128
   
   FIX: Run this PowerShell command (requires AD admin):
   
   Set-ADUser -Identity 'sqlsvc' -Replace @{'msDS-SupportedEncryptionTypes'=28}
   
   This adds AES128, AES256, and RC4 support to the service account.
   
   📖 Documentation: https://learn.microsoft.com/sql/...
```
