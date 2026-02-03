# Feature Specification: SQL Server Connection Troubleshooting

**Feature Branch**: `001-troubleshooting`  
**Created**: 2026-02-02  
**Status**: Draft  
**Input**: Requirements gathered from CSS support team meetings (Oct 2025, Feb 2026) regarding common SQL Server connection issues

## Overview

This feature adds a comprehensive, **extensible troubleshooting framework** to go-sqlcmd. The first implementation focuses on **connectivity and driver diagnostics**, but the architecture supports future troubleshooting types.

### Command Structure

```
sqlcmd troubleshoot <type> [flags]

Types:
  connectivity    Diagnose connection, authentication, and driver issues (Phase 1)
  vm-config       Azure SQL VM configuration analysis (Future)
  performance     Query and server performance diagnostics (Future)
  replication     AG/replication connectivity issues (Future)
```

**Phase 1 Scope**: `sqlcmd troubleshoot connectivity` - everything in this spec relates to connectivity/driver troubleshooting.

### Connectivity Troubleshooting Capabilities

The connectivity troubleshooter provides automated analysis similar to what CSS support engineers perform manually:

- **System information collection** (like SQL Check)
- **Connectivity testing** (like portqry)
- **Driver diagnostic tracing** (BID tracing)
- **Network tracing** (using built-in OS tools: netsh/tcpdump)
- **Kerberos/SSPI diagnostics**
- **Certificate validation**
- **Log analysis**

Key design principle from CSS: *"Having it all done within the tool - data collection, tracing, connection test - in one command would make life so much easier for customers and us from support."*

### Problem Statement

Current troubleshooting workflow is painful:
1. CSS uses unsigned PowerShell scripts (SQL Check, SQL Trace) - many customers can't run them
2. Linux has very limited diagnostic tooling
3. Collecting data requires multiple manual steps that customers often get wrong
4. Tools like Kerberos Configuration Manager are no longer maintained
5. Engineers spend 10-15 minutes walking customers through setup that could take seconds

### Success Vision

> "You take something that takes a few minutes to walk a customer through or make them type each command... and then they execute. So you take at least 10-15 minutes minimum down to just a few seconds." - Shon Hauck

### Extensible Framework Architecture

The `troubleshoot` command is designed as an extensible parent command with subcommands for different troubleshooting types:

```
sqlcmd troubleshoot
├── connectivity    # Phase 1 - This spec
├── vm-config       # Future - Azure SQL VM best practices
├── performance     # Future - Query/server perf analysis
└── replication     # Future - AG/DR connectivity
```

**Design Principles:**
1. Each troubleshooting type is a separate subcommand with its own flags
2. Common infrastructure (output formatting, redaction, consent prompts) is shared
3. New troubleshooting types can be added without modifying existing commands
4. Default behavior (no subcommand) shows available troubleshooting types

**Future Troubleshooting Types (Out of Scope for Phase 1):**
- `vm-config` - Azure SQL VM configuration validation (IaaS best practices, storage, networking)
- `performance` - Wait stats analysis, blocking detection, query plan issues
- `replication` - AG listener connectivity, endpoint configuration, certificate auth between replicas

---

## User Scenarios & Testing *(mandatory)*

<!--
  User stories are prioritized as user journeys ordered by importance.
  Each user story/journey is independently testable - meaning if you implement just ONE of them,
  you should still have a viable MVP that delivers value.
-->

**Note**: All user stories below are for `sqlcmd troubleshoot connectivity` (Phase 1 scope).

### User Story 1 - Basic Connectivity Troubleshooting (Priority: P1)

As a database administrator, I want to test basic network connectivity to a SQL Server instance so that I can quickly determine if the server is reachable and listening on the expected port.

**Why this priority**: Network connectivity is the most fundamental requirement for any SQL Server connection. If the server isn't reachable, no other troubleshooting matters. This provides immediate value similar to portqry functionality.

**Independent Test**: Can be fully tested by running `sqlcmd troubleshoot connectivity --server <servername> --port <port>` and receiving a clear pass/fail result with diagnostic information.

**Acceptance Scenarios**:

1. **Given** a SQL Server hostname and port, **When** the user runs connectivity test, **Then** the tool reports whether the port is open and reachable
2. **Given** a SQL Server that is not listening, **When** the user runs connectivity test, **Then** the tool provides clear error message with suggestions (firewall, service not running, wrong port)
3. **Given** a SQL Server with multiple instances, **When** the user provides instance name, **Then** the tool resolves the dynamic port via SQL Browser and tests connectivity

---

### User Story 2 - Connection String Validation (Priority: P1)

As a developer, I want to provide my application's connection string to sqlcmd and have it validate all aspects of the connection, telling me exactly what's wrong and how to fix it.

**Why this priority**: This is the core troubleshooting workflow requested by the support team - users get an error and want to know why their connection string doesn't work.

**Independent Test**: Can be fully tested by running `sqlcmd troubleshoot connectivity --connection-string "<connstring>"` and receiving a detailed analysis report.

**Acceptance Scenarios**:

1. **Given** a valid connection string, **When** the user runs troubleshoot, **Then** the tool validates each component and reports success
2. **Given** a connection string with authentication issues, **When** the user runs troubleshoot, **Then** the tool identifies the specific authentication failure and provides remediation steps
3. **Given** a connection string with TLS/certificate issues, **When** the user runs troubleshoot, **Then** the tool identifies certificate problems and suggests fixes
4. **Given** a connection string with Kerberos issues, **When** the user runs troubleshoot, **Then** the tool identifies SSPI/Kerberos problems and provides PowerShell commands to fix

---

### User Story 3 - Kerberos/SSPI Diagnostics (Priority: P1)

As a database administrator experiencing "Cannot generate SSPI context" errors, I want sqlcmd to analyze my Kerberos configuration and tell me exactly what's misconfigured.

**Why this priority**: According to the CSS team, this is "the bulk of our generic Kerberos errors" and a very common support case. The existing Kerberos Configuration Manager tool does not check encryption levels.

**Independent Test**: Can be fully tested by running `sqlcmd troubleshoot connectivity --kerberos --server <servername>` and receiving encryption type analysis and SPN validation.

**Acceptance Scenarios**:

1. **Given** a service account with RC4 only and client requiring AES, **When** the user runs Kerberos diagnostics, **Then** the tool detects the encryption type mismatch and provides the PowerShell command to fix it (`Set-ADUser -KerberosEncryptionType`)
2. **Given** duplicate SPNs in Active Directory, **When** the user runs Kerberos diagnostics, **Then** the tool identifies the duplicate SPNs and shows which accounts have them
3. **Given** an SPN registered on the wrong account, **When** the user runs Kerberos diagnostics, **Then** the tool identifies the mismatch and suggests the correct account
4. **Given** proper Kerberos configuration, **When** the user runs Kerberos diagnostics, **Then** the tool reports all checks passed

---

### User Story 4 - Certificate/TLS Validation (Priority: P2)

As a database administrator with TLS encryption enabled, I want sqlcmd to validate that my SQL Server certificate is properly configured for all connection scenarios.

**Why this priority**: Certificate issues are common when forcing encryption, especially with listeners, short names vs FQDNs, and Always On configurations.

**Independent Test**: Can be fully tested by running `sqlcmd troubleshoot connectivity --certificate --server <servername>` and receiving certificate analysis.

**Acceptance Scenarios**:

1. **Given** a connection using short name and certificate only has FQDN in SAN, **When** the user runs certificate validation, **Then** the tool identifies the SAN mismatch and explains the fix
2. **Given** an AG listener name not in the certificate SAN, **When** the user runs certificate validation, **Then** the tool identifies the missing listener name
3. **Given** force encryption enabled but certificate missing private key permissions for service account, **When** the user runs certificate validation, **Then** the tool identifies the permission issue
4. **Given** proper certificate configuration, **When** the user runs certificate validation, **Then** the tool reports certificate is valid for the connection name

---

### User Story 5 - Log Analysis (Priority: P2)

As a database administrator troubleshooting connection issues, I want sqlcmd to scan SQL Server logs and system logs to extract all errors and warnings related to my connection problem.

**Why this priority**: Log analysis is a fundamental troubleshooting step that currently requires manual effort across multiple log files.

**Independent Test**: Can be fully tested by running `sqlcmd troubleshoot connectivity --logs --server <servername>` and receiving categorized error/warning output.

**Acceptance Scenarios**:

1. **Given** SQL Server error log access, **When** the user runs log analysis, **Then** the tool extracts and categorizes connection-related errors and warnings
2. **Given** Windows Event Log access (Windows only), **When** the user runs log analysis, **Then** the tool correlates SQL and system events
3. **Given** a time range filter, **When** the user runs log analysis, **Then** the tool only returns errors within that timeframe
4. **Given** macOS or Linux, **When** the user runs log analysis, **Then** the tool analyzes appropriate system logs (syslog, journald)

---

### User Story 6 - System Information Collection (Priority: P1)

As a support engineer, I want sqlcmd to collect comprehensive system information similar to SQL Check, so I can quickly identify configuration issues.

**Why this priority**: SQL Check is the primary CSS tool but is an unsigned PowerShell script many customers can't run. Having this in sqlcmd solves the deployment problem.

**Independent Test**: Can be fully tested by running `sqlcmd troubleshoot connectivity --sysinfo` and receiving a comprehensive system report.

**Acceptance Scenarios**:

1. **Given** a Windows machine, **When** the user runs sysinfo collection, **Then** the tool reports: OS version, computer name, domain info, user context, TLS versions enabled/disabled, cipher suites, ODBC/OLEDB drivers installed
2. **Given** a Linux machine, **When** the user runs sysinfo collection, **Then** the tool reports: OS version, hostname, ODBC drivers, ODBC INI settings, krb5.conf settings
3. **Given** the `--who-am-i` flag, **When** the user runs sysinfo, **Then** the tool shows current user context and security identity being used for authentication

---

### User Story 7 - Driver Diagnostic Tracing (Priority: P1)

As a support engineer, I want sqlcmd to automatically enable driver diagnostic tracing (BID trace), run a connection test, and collect the trace - all in one command.

**Why this priority**: Currently requires multiple manual steps, registry changes, and process recycle. CSS engineers request this constantly.

**Independent Test**: Can be fully tested by running `sqlcmd troubleshoot connectivity --trace --server <servername>` and receiving driver diagnostic logs.

**Acceptance Scenarios**:

1. **Given** a Windows machine, **When** the user runs trace collection, **Then** BID tracing is enabled, connection attempted, trace stopped, and ETL file produced
2. **Given** a Linux machine, **When** the user runs trace collection, **Then** ODBC tracing is enabled via INI, connection attempted, and trace file produced
3. **Given** the trace output, **When** analyzing connection issues, **Then** the trace shows socket open, DNS resolution, SSL/TLS handshake, and login phase details

---

### User Story 8 - Network Trace Collection (Priority: P2)

As a support engineer, I want sqlcmd to capture network traces during a connection test using built-in OS tools, without requiring Wireshark or other installs.

**Why this priority**: Network traces correlated with driver traces prove where issues occur. Using built-in tools (netsh/tcpdump) means no additional software needed.

**Independent Test**: Can be fully tested by running `sqlcmd troubleshoot connectivity --nettrace --server <servername>` and receiving a network capture file.

**Acceptance Scenarios**:

1. **Given** a Windows machine with admin rights, **When** the user runs network trace, **Then** netsh trace captures packets filtered to SQL ports, produces ETL file
2. **Given** a Linux machine with appropriate permissions, **When** the user runs network trace, **Then** tcpdump captures packets filtered to SQL ports, produces PCAP file
3. **Given** both `--trace` and `--nettrace` flags, **When** the user runs combined tracing, **Then** both BID trace and network trace are collected simultaneously and correlated

---

### User Story 9 - Retry/Stress Testing (Priority: P2)

As a database administrator with intermittent connection issues, I want to run multiple connection attempts with configurable delay to reproduce and diagnose intermittent failures.

**Why this priority**: Intermittent issues are common and hard to reproduce. Having a built-in retry mechanism makes diagnosis much easier.

**Independent Test**: Can be fully tested by running `sqlcmd troubleshoot connectivity --retry 100 --delay 500 --server <servername>`.

**Acceptance Scenarios**:

1. **Given** retry count and delay parameters, **When** the user runs retry test, **Then** the tool connects N times with specified delay, reports success/failure for each
2. **Given** a failure during retry test, **When** the failure occurs, **Then** the tool logs the specific error, attempt number, and timing
3. **Given** `--stop-on-failure` flag, **When** a connection fails, **Then** the test stops and reports details of the failure

---

### User Story 10 - Comprehensive Troubleshooting Report (Priority: P2)

As a support engineer, I want sqlcmd to generate a comprehensive diagnostic report that I can share with customers or attach to support cases.

**Why this priority**: The CSS team currently uses SQLCheck to gather this information. Having it built into sqlcmd streamlines the troubleshooting workflow.

**Independent Test**: Can be fully tested by running `sqlcmd troubleshoot connectivity --full-report --server <servername> --output report.zip` and receiving a complete diagnostic package.

**Acceptance Scenarios**:

1. **Given** a server connection, **When** the user runs full report, **Then** the tool generates separate files for each data type (sysinfo, traces, logs) and compresses into ZIP/CAB
2. **Given** the `--output` flag, **When** the user specifies a file path, **Then** the report package is saved to that location
3. **Given** the `--redact` flag, **When** the user wants to share the report, **Then** sensitive information (passwords, specific IPs) is removed
4. **Given** report completion, **When** output is generated, **Then** the tool provides instructions for uploading to Microsoft File Share

---

### User Story 11 - Connection Phase Analysis (Priority: P1)

As a database administrator, I want sqlcmd to show me exactly where time is being spent during connection so I know whether the problem is client-side, network, or SQL Server.

**Why this priority**: This routing information is critical - if the issue is SQL Server performance, troubleshooting the client is wasted effort.

**Independent Test**: Can be fully tested by running `sqlcmd troubleshoot connectivity --timing --server <servername>` and seeing connection phase breakdown.

**Acceptance Scenarios**:

1. **Given** a connection attempt, **When** the user runs timing analysis, **Then** the tool shows time spent in: DNS resolution, TCP socket open, SSL/TLS handshake, login phase
2. **Given** slow login phase with fast earlier phases, **When** analysis completes, **Then** the tool suggests "This appears to be a SQL Server performance issue, not a client connectivity issue"
3. **Given** slow SSL/TLS handshake, **When** analysis completes, **Then** the tool suggests cipher suite or certificate issues to investigate

---

### User Story 12 - Listener Mode for Production Testing (Priority: P3 - Stretch Goal)

As a support engineer, I want to test network connectivity without involving SQL Server, so I can diagnose during production without impacting the running application.

**Why this priority**: Stretch goal - very powerful for CSS but requires significant engineering effort. Allows testing during peak production times without process recycle.

**Independent Test**: Run `sqlcmd listen --port 14330` on target, then `sqlcmd troubleshoot connectivity --listener-test --server target --port 14330` from client.

**Acceptance Scenarios**:

1. **Given** sqlcmd running in listen mode on target machine, **When** client runs listener test, **Then** connection is established without SQL Server involvement, proving network path works
2. **Given** bidirectional listener test, **When** both sides try to connect to each other, **Then** report shows where in the network path connectivity breaks
3. **Given** firewall blocking traffic, **When** listener test fails, **Then** tool identifies that listener never received the connection attempt

---

### User Story 13 - Delegation Settings Validation (Priority: P2)

As a database administrator using Kerberos delegation (double-hop scenarios), I want sqlcmd to validate that delegation is properly configured.

**Why this priority**: Delegation issues are a major source of support calls according to CSS, second only to SPN issues.

**Independent Test**: Can be fully tested by running `sqlcmd troubleshoot connectivity --kerberos --check-delegation --server <servername>`.

**Acceptance Scenarios**:

1. **Given** a service account, **When** the user runs delegation check, **Then** the tool reports whether delegation is enabled and what type (unconstrained, constrained, resource-based)
2. **Given** constrained delegation, **When** the user runs delegation check, **Then** the tool shows which services the account is trusted to delegate to
3. **Given** missing delegation configuration, **When** trying a double-hop scenario, **Then** the tool identifies delegation as the likely issue and provides fix commands

---

### User Story 14 - TDS Protocol Parsing (Priority: P3 - Phase 2)

As a support engineer analyzing network captures, I want sqlcmd to parse TDS protocol packets so I can understand the connection handshake without using Wireshark plugins or external tools.

**Why this priority**: Phase 2 feature - requires significant engineering effort. TDS View was valuable but "fell off the radar" per CSS feedback.

**Independent Test**: Run `sqlcmd troubleshoot connectivity --parse-tds <capture.pcap>` to see human-readable TDS analysis.

**Acceptance Scenarios**:

1. **Given** a network capture file (pcap/etl), **When** the user runs TDS parsing, **Then** the tool displays prelogin, login7, SSPI token exchange, and response packets in human-readable format
2. **Given** a failed connection capture, **When** TDS parsing runs, **Then** the tool highlights where in the TDS handshake the failure occurred
3. **Given** an encrypted TDS stream, **When** parsing, **Then** the tool indicates encryption started and shows what was visible before encryption

---

### Edge Cases

- What happens when the user doesn't have permissions to query Active Directory for SPN/encryption information?
- How does the tool handle SQL Server instances behind a load balancer?
- What happens when connecting to Azure SQL Database vs on-premises SQL Server?
- How does the tool behave when SQL Browser service is not running?
- What happens when the user specifies both `--connection-string` and individual parameters?
- How does the tool handle air-gapped environments with no internet access?
- What happens when netsh/tcpdump require elevated permissions the user doesn't have?

---

## Requirements *(mandatory)*

### Functional Requirements

**System Information Collection (SQL Check equivalent)**

- **FR-001**: System MUST collect OS version, edition, computer name
- **FR-002**: System MUST collect domain information and current user context
- **FR-003**: System MUST report TLS versions enabled/disabled in registry (Windows)
- **FR-004**: System MUST report cipher suites and protocol order
- **FR-005**: System MUST list all ODBC drivers installed with version info
- **FR-006**: System MUST list all OLEDB providers installed (Windows)
- **FR-007**: System MUST report "disabled loopback check" setting
- **FR-008**: System MUST dump ODBC INI settings (Linux)
- **FR-009**: System MUST dump DSN parameters being used
- **FR-010**: System MUST show sqlcmd/driver version (major.minor)

**Connectivity Testing**

- **FR-011**: System MUST test TCP connectivity to the specified SQL Server port
- **FR-012**: System MUST resolve SQL Server instance names via SQL Browser service (UDP 1434)
- **FR-013**: System MUST detect if the SQL Server port is blocked by firewall
- **FR-014**: System MUST report network latency to the server
- **FR-015**: System MUST support testing named pipes connectivity (Windows)
- **FR-016**: System MUST flush DNS cache before connectivity test (configurable)
- **FR-017**: System MUST flush Kerberos tickets before test (configurable)

**Connection String Analysis**

- **FR-020**: System MUST parse and validate connection string syntax
- **FR-021**: System MUST identify the driver type specified in the connection string
- **FR-022**: System MUST test each component of the connection (network, auth, encryption)
- **FR-023**: System MUST provide specific error messages for each failure type
- **FR-024**: System MUST suggest fix commands (PowerShell, CLI) for identified issues
- **FR-025**: System MUST flag non-default settings as potential issues (high timeouts, etc.)
- **FR-026**: System MUST warn when Connection Timeout > 60 seconds ("Band-Aid fix detected")

**Connection Phase Timing**

- **FR-030**: System MUST measure and report time spent in DNS resolution
- **FR-031**: System MUST measure and report time spent in TCP socket open
- **FR-032**: System MUST measure and report time spent in SSL/TLS handshake
- **FR-033**: System MUST measure and report time spent in login phase
- **FR-034**: System MUST identify which phase is the bottleneck
- **FR-035**: System MUST detect TNIR (Transparent Network IP Resolution) behavior and explain timeout segmentation

**Driver Diagnostic Tracing (BID Trace)**

- **FR-040**: System MUST enable BID tracing on Windows via registry
- **FR-041**: System MUST enable ODBC tracing on Linux via INI settings
- **FR-042**: System MUST automatically start tracing before connection attempt
- **FR-043**: System MUST automatically stop tracing after connection attempt
- **FR-044**: System MUST output trace to specified file location
- **FR-045**: System MUST NOT require process recycle (since SQLCMD is fresh process)

**Network Tracing**

- **FR-050**: System MUST use netsh on Windows for network trace (no additional tools)
- **FR-051**: System MUST use tcpdump on Linux for network trace (no additional tools)
- **FR-052**: System MUST filter network trace to relevant ports (1433, 1434, specified port)
- **FR-053**: System MUST correlate network trace timing with driver trace
- **FR-054**: System MUST output ETL (Windows) or PCAP (Linux) file

**Retry/Stress Testing**

- **FR-060**: System MUST support configurable number of connection attempts
- **FR-061**: System MUST support configurable delay between attempts (milliseconds)
- **FR-062**: System MUST report success/failure for each attempt
- **FR-063**: System MUST support --stop-on-failure mode
- **FR-064**: System MUST collect traces during retry test if requested

**Kerberos/SSPI Diagnostics**

- **FR-070**: System MUST query Active Directory for service account encryption types (msDS-SupportedEncryptionTypes)
- **FR-071**: System MUST query local machine allowed encryption types from security policy
- **FR-072**: System MUST detect encryption type mismatches between service account and client
- **FR-073**: System MUST search for SPNs (MSSQLSvc) and detect duplicates
- **FR-074**: System MUST identify if SPN is registered on wrong account
- **FR-075**: System MUST detect if Kerberos ticket has already been obtained and cached
- **FR-076**: System MUST provide PowerShell commands to fix identified Kerberos issues
- **FR-077**: System MUST support GMSA (Group Managed Service Accounts) analysis
- **FR-078**: System MUST check DNS suffix used in TGS request
- **FR-079**: System MUST validate Kerberos delegation settings (unconstrained, constrained, resource-based)
- **FR-080**: System MUST show which services an account is trusted to delegate to

**Certificate/TLS Validation**

- **FR-082**: System MUST retrieve the certificate thumbprint configured for SQL Server
- **FR-083**: System MUST validate certificate SAN contains the connection name
- **FR-084**: System MUST check certificate expiration status
- **FR-085**: System MUST verify service account has read permission on certificate private key
- **FR-086**: System MUST detect Force Encryption setting on the SQL Server instance
- **FR-087**: System MUST validate certificate chain and trust
- **FR-088**: System MUST detect known cipher issues (Diffie-Hellman zero-padding bug)
- **FR-089**: System MUST detect cipher suite typos in policy configuration
- **FR-090**: System MUST identify "target principal name is incorrect" as certificate issue (not just Kerberos)

**Network Trace Analysis**

- **FR-091**: System MUST detect SYN-ACK-RESET pattern indicating server rejection
- **FR-092**: System MUST analyze TTL values to detect asymmetric routing/multi-pathing
- **FR-093**: System MUST identify NTLM vs Kerberos authentication from trace
- **FR-094**: System MUST detect NTLM performance issues (1-second DC timeout)

**Listener Mode (Stretch Goal)**

- **FR-095**: System MUST support running as a TCP listener on specified port
- **FR-096**: System MUST support listener-test mode to test connectivity to another sqlcmd listener
- **FR-097**: System MUST support bidirectional listener testing to identify network break points
- **FR-098**: System MUST operate without SQL Server involvement for production-safe testing

**Log Analysis**

- **FR-100**: System MUST read SQL Server error log (ERRORLOG files)
- **FR-101**: System MUST filter log entries by severity (Error, Warning)
- **FR-102**: System MUST filter log entries by time range
- **FR-103**: System MUST categorize connection-related errors
- **FR-104**: System MUST read Windows Event Log (Application, System, Security) on Windows
- **FR-105**: System MUST read syslog/journald on Linux
- **FR-106**: System MUST read system.log and appropriate logs on macOS

**Performance Safety & User Consent**

- **FR-PF-001**: System MUST NOT execute any operation that could impact production performance without explicit user consent
- **FR-PF-002**: System MUST categorize all operations into safety tiers:
  - **Tier 1 (Safe)**: Read-only metadata queries, local system info - no consent required
  - **Tier 2 (Low Impact)**: Lightweight DMV queries (dm_exec_connections, dm_exec_sessions) - consent via `--query-server` flag
  - **Tier 3 (Moderate Impact)**: Aggregate DMVs (dm_os_wait_stats), catalog views - consent + warning displayed
  - **Tier 4 (High Impact)**: XEvents, ring buffers, network tracing - explicit `--allow-impact` flag required
- **FR-PF-003**: System MUST display warning before Tier 3+ operations: "This operation may impact server performance. Continue? [y/N]"
- **FR-PF-004**: System MUST support `--dry-run` flag to show what would be collected without executing
- **FR-PF-005**: System MUST support `--yes` or `-y` flag to auto-confirm all prompts (for scripting)
- **FR-PF-006**: System MUST use minimal, targeted queries - only SELECT the specific columns needed, never `SELECT *`
- **FR-PF-006a**: System MUST filter DMV queries to only relevant rows (e.g., current session, specific wait types, connection-related only)
- **FR-PF-006b**: System MUST avoid joins on large DMVs; prefer multiple small targeted queries over complex joins
- **FR-PF-007**: System MUST limit result sets from DMVs (e.g., TOP 100) to prevent memory issues
- **FR-PF-008**: System MUST timeout individual server queries (default: 10 seconds) to prevent blocking
- **FR-PF-009**: System MUST display estimated impact level before each server-side operation
- **FR-PF-010**: System MUST skip server-side diagnostics entirely with `--client-only` flag
- **FR-PF-011**: System MUST default to client-only diagnostics (no server queries) unless `--query-server` is specified
- **FR-PF-012**: System MUST log all server-side queries executed for audit purposes

**Server-Side SQL Diagnostics (when connected with --query-server)**

- **FR-107**: System MUST query connection-related DMVs when successfully connected AND `--query-server` flag specified:
  - `sys.dm_exec_connections` - active connection info, auth scheme, encryption (Tier 2)
  - `sys.dm_exec_sessions` - session info, login name, host info (Tier 2)
  - `sys.dm_os_wait_stats` - wait types affecting connections (Tier 3 - requires consent)
  - `sys.dm_exec_requests` - pending requests, blocking (Tier 2)
- **FR-107a**: System MUST query `sys.dm_exec_connections` for auth_scheme to confirm Kerberos vs NTLM
- **FR-107b**: System MUST query `sys.dm_exec_connections` for encrypt_option to confirm TLS status
- **FR-107c**: System MUST display warning: "Querying server DMVs - minimal performance impact expected" before Tier 2 queries
- **FR-108**: System MUST query catalog views for server configuration (Tier 2):
  - `sys.configurations` - relevant server settings (remote login timeout, network packet size)
  - `sys.endpoints` - endpoint configuration for TDS
  - `sys.certificates` - certificate info if accessible
  - `sys.credentials` - credential mappings (names only, not secrets)
- **FR-108a**: System MUST query `sys.server_principals` for login configuration
- **FR-108b**: System MUST query `sys.linked_servers` for linked server config if delegation issues suspected
- **FR-109**: System MUST capture Extended Events for connection diagnostics ONLY when BOTH `--xevent` AND `--allow-impact` flags specified:
  - `sqlserver.login` - login events with client info
  - `sqlserver.logout` - disconnect events
  - `sqlserver.error_reported` - errors including auth failures
  - `sqlserver.connectivity_ring_buffer_recorded` - connection ring buffer events
  - `sqlserver.prelogin_trace` - prelogin packet info (if available)
- **FR-109a**: System MUST create temporary XEvent session, capture during test, then drop session
- **FR-109b**: System MUST support `--xevent-duration <seconds>` to limit capture time (default: 30 seconds, max: 300 seconds)
- **FR-109c**: System MUST query `sys.dm_xe_session_targets` for existing connectivity sessions (Tier 2)
- **FR-109d**: System MUST query `sys.dm_os_ring_buffers` for RING_BUFFER_CONNECTIVITY and RING_BUFFER_SECURITY_ERROR (Tier 3 - requires consent)
- **FR-109e**: System MUST display prominent warning before XEvent capture: "WARNING: Creating XEvent session may impact production performance. Recommended for non-production or during maintenance windows."
- **FR-109f**: System MUST verify XEvent session was successfully dropped, and warn user if cleanup fails

**Reporting & Output**

- **FR-110**: System MUST output results in human-readable text format by default (for terminal/stdout)
- **FR-110a**: System MUST default to Markdown format when `--output` is specified without explicit format/extension
- **FR-111**: System MUST support JSON output format for automation (`--output-format json` or `.json` extension)
- **FR-112**: System MUST support HTML report format for sharing (`--output-format html` or `.html` extension)
- **FR-112a**: System MUST support Markdown output format for AI tool integration, VS Code rendering, and Word document conversion (`--output-format md` or `.md` extension)
- **FR-113**: System MUST support redacting sensitive information in reports
- **FR-114**: System MUST include remediation steps in all output formats
- **FR-115**: System MUST package all collected files into ZIP/CAB archive
- **FR-116**: System MUST provide upload instructions for Microsoft File Share

**Sensitive Data Filtering (Air-Gapped/Secure Environments)**

- **FR-117**: System MUST support `--no-sensitive` flag to prevent collection of potentially sensitive data
- **FR-118**: System MUST support `--redact-patterns` flag to specify custom redaction patterns (regex)
- **FR-119**: System MUST redact passwords, connection strings with credentials, and auth tokens by default when `--no-sensitive` is set
- **FR-119a**: System MUST skip collection of network packet payloads when `--no-sensitive` is set
- **FR-119b**: System MUST redact usernames and server names when `--redact-hostnames` is specified
- **FR-119c**: System MUST display summary of what data types were excluded/redacted
- **FR-119d**: System MUST allow preview of collected data before packaging with `--preview` flag
- **FR-119e**: System MUST support `--redact-config <file>` to load redaction rules from config file

**Advanced/Expert Mode**

- **FR-120**: System MUST support `--advanced` flag to enable expert-level diagnostics (shown separately in help as "Advanced Options")
- **FR-120a**: Advanced options MUST be hidden from default `--help` output but visible with `--help-advanced` or `--help-all`
- **FR-120b**: Advanced options MUST display warnings when used (e.g., "Advanced mode: This may generate large trace files and detailed output")
- **FR-121**: System MUST support verbose output for screen-sharing diagnostics (`--verbose` or `-v`)
- **FR-122**: System MUST allow collecting deeper data sets for escalation scenarios (`--deep-collect`)

**Graceful Degradation (Decision 1 & 2)**

- **FR-123**: System MUST detect AD accessibility at startup before running Kerberos diagnostics
- **FR-124**: System MUST display warning when AD queries are unavailable (e.g., "AD queries unavailable - some Kerberos diagnostics will be skipped")
- **FR-125**: System MUST continue with non-AD-dependent checks when AD is unavailable
- **FR-126**: System MUST detect whether running locally on SQL Server or remotely
- **FR-127**: System MUST clearly indicate which checks cannot be performed remotely
- **FR-128**: System MUST suggest running locally for complete diagnostics (e.g., "For complete diagnostics, run: sqlcmd troubleshoot connectivity --server localhost on the SQL Server machine")

**Azure SQL Platform Support (Decision 3)**

- **FR-129**: System MUST detect Azure SQL Database endpoints and adjust available diagnostics
- **FR-130**: System MUST validate Entra ID (AAD) authentication for Azure platforms
- **FR-131**: System MUST check Azure SQL firewall rules when connection fails
- **FR-132**: System MUST detect connection policy (Redirect vs Proxy) for Azure SQL Database
- **FR-133**: System MUST validate Azure SQL Managed Instance VNet connectivity
- **FR-134**: System MUST check MI public endpoint configuration when applicable
- **FR-135**: System MUST validate SQL Database in Fabric endpoint format and workspace permissions
- **FR-136**: System MUST detect Synapse DW pause/resume state
- **FR-137**: System MUST differentiate Synapse dedicated vs serverless endpoints
- **FR-138**: System MUST validate Fabric DW endpoint format and connectivity
- **FR-139**: System MUST display clear messaging when checks don't apply (e.g., "Windows Auth not supported on Azure SQL Database")

**Guided Credential Flow (Decision 4)**

- **FR-140**: System MUST prompt user for appropriate credentials with explanation of why each is needed
- **FR-141**: System MUST guide user through signing in as the user experiencing the issue for Windows Auth testing
- **FR-142**: System MUST prompt for domain credentials when AD queries are needed for SPN/delegation info
- **FR-143**: System MUST support separate credentials for AD queries vs SQL Server connection

**Privilege Elevation (Decision 5)**

- **FR-144**: System MUST detect when network tracing requires elevated privileges
- **FR-145**: System MUST prompt for runas/sudo elevation with explanation
- **FR-146**: System MUST allow user to decline elevation and continue with best-effort diagnostics
- **FR-147**: System MUST display message when continuing without elevation (e.g., "Continuing without network trace - some diagnostics will be limited")

**Trace File Management (Decision 6)**

- **FR-148**: System MUST support `--trace-max-size` flag to limit per-file trace size (default: 50MB)
- **FR-149**: System MUST support `--trace-max-files` flag to set rotation count (default: 5)
- **FR-150**: System MUST support `--trace-mode` flag with values `circular` or `sequential` (default: sequential)
- **FR-151**: System MUST auto-rotate trace files when max size is reached
- **FR-152**: System MUST delete oldest trace file when max file count is exceeded in rotation

**TDS Protocol Parsing (Decision 7 - Phase 2)**

- **FR-153**: System MUST parse TDS prelogin packets showing version, encryption, instance name
- **FR-154**: System MUST parse TDS login7 packets showing client info, database, options
- **FR-155**: System MUST parse SSPI token exchange packets
- **FR-156**: System MUST parse TDS error and info messages
- **FR-157**: System MUST indicate where TDS encryption begins in the stream
- **FR-158**: System MUST accept pcap and etl capture file formats

**Architecture: Data Provider Abstraction**

- **FR-159**: System MUST implement a `DiagnosticDataProvider` interface separating data collection from analysis
- **FR-159a**: System MUST support `LiveSQLProvider` for querying real SQL Server instances
- **FR-159b**: System MUST support `FileProvider` for reading pre-collected diagnostic data from files (JSON/XML)
- **FR-159c**: System MUST support `--from-file <path>` flag to analyze previously-collected diagnostic data
- **FR-159d**: System MUST support `--save-data <path>` flag to save collected raw data for later analysis
- **FR-160**: System MUST enable unit testing of analysis logic without requiring live SQL Server
- **FR-160a**: System MUST include pre-generated test data files for common scenarios (Kerberos mismatch, cert issues, etc.)
- **FR-161**: System SHOULD support `--simulate <scenario>` flag for AI-generated test data (Phase 2+)
- **FR-161a**: Simulation scenarios SHOULD include: "high wait states", "deadlocks", "auth failures", "encryption mismatches"

**Architecture: Extensible Troubleshooting Framework**

- **FR-162**: System MUST implement `troubleshoot` as a parent command with subcommands for different troubleshooting types
- **FR-162a**: System MUST implement `connectivity` as the first subcommand (Phase 1 scope)
- **FR-162b**: Parent `troubleshoot` command with no subcommand MUST display available troubleshooting types and usage
- **FR-163**: System MUST share common infrastructure across troubleshooting types:
  - Output formatting (text, JSON, Markdown, HTML)
  - Consent/warning prompts
  - Redaction and sensitive data handling
  - Progress indication
- **FR-163a**: Each troubleshooting subcommand MUST be independently implementable without modifying other subcommands
- **FR-164**: System MUST define a `Troubleshooter` interface that all troubleshooting types implement
- **FR-164a**: `Troubleshooter` interface MUST include: `Run()`, `GetResults()`, `GetRemediations()`

### Non-Functional Requirements

- **NFR-001**: System MUST work on Windows, macOS, and Linux
- **NFR-002**: System MUST NOT require elevated/administrator privileges for basic connectivity tests
- **NFR-003**: System MUST clearly indicate when elevated privileges are required for specific checks
- **NFR-004**: System MUST complete basic connectivity test within 30 seconds
- **NFR-005**: System MUST handle timeouts gracefully with configurable timeout values
- **NFR-006**: System MUST provide progress indication for long-running operations
- **NFR-007**: System MUST support localization through existing localizer package
- **NFR-008**: System MUST NOT require installation of additional tools (Wireshark, Netmon, etc.)
- **NFR-009**: System MUST work in air-gapped environments (no internet required)
- **NFR-010**: System MUST produce unsigned-script-free diagnostics (solving CSS's main deployment problem)
- **NFR-011**: Network/driver traces MUST be filterable to minimize noise and file size
- **NFR-012**: System MUST NOT impact production SQL Server performance without explicit user consent
- **NFR-013**: System MUST default to safe, client-only operations (opt-in for server queries)
- **NFR-014**: System MUST timeout all server-side queries within configurable limit (default: 10 seconds)
- **NFR-015**: System MUST provide clear warnings before any potentially impactful operation
- **NFR-016**: System MUST support fully non-interactive mode for scripting with `--yes` flag

**Extensibility**

- **NFR-026**: Framework MUST allow adding new troubleshooting types without modifying existing code
- **NFR-027**: Common infrastructure (output, consent, redaction) MUST be reusable across troubleshooting types
- **NFR-028**: Each troubleshooting type MUST be independently testable

**Code Quality & Compatibility**

- **NFR-017**: Command MUST follow existing go-sqlcmd Cobra command patterns (see `cmd/modern/root/` for examples)
- **NFR-018**: Command MUST use `cmdparser.Cmd` embedding and `DefineCommand` pattern per project conventions
- **NFR-019**: Command MUST NOT modify or break any existing sqlcmd commands, flags, or behaviors
- **NFR-020**: Command MUST use existing `internal/` packages where applicable (localizer, output, config, etc.)
- **NFR-021**: Command MUST follow existing error handling patterns using `localizer.Errorf`
- **NFR-022**: All new user-facing strings MUST use localizer package for internationalization
- **NFR-023**: Command MUST include unit tests following existing `*_test.go` patterns
- **NFR-024**: Command MUST pass `golangci-lint` with no new warnings
- **NFR-025**: Command MUST maintain backward compatibility - no changes to existing CLI contract

### Key Entities

- **TroubleshootResult**: Overall result containing all diagnostic checks and their outcomes
- **ConnectivityCheck**: Result of network/port connectivity test
- **KerberosCheck**: Result of Kerberos/SSPI configuration analysis
- **CertificateCheck**: Result of TLS certificate validation
- **LogEntry**: Parsed log entry with severity, timestamp, message, and category
- **Remediation**: Suggested fix with description, command, and documentation link

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Users can diagnose basic connectivity issues in under 30 seconds
- **SC-002**: Users can identify Kerberos encryption mismatches without manual Active Directory queries
- **SC-003**: Users can validate certificate configuration without manually inspecting certificate stores
- **SC-004**: Users can generate a complete diagnostic report with a single command
- **SC-005**: 90% of common "Cannot generate SSPI context" errors can be diagnosed and have remediation provided
- **SC-006**: 90% of common certificate/TLS connection errors can be diagnosed with specific fix suggestions
- **SC-007**: Reduce typical CSS Kerberos troubleshooting time by 50% compared to manual process

---

## Technical Considerations

### Architecture: Data Collection vs Analysis Separation

**Design Principle**: Put an abstraction layer between the part that queries the database/server and the part that analyzes the data.

```
┌─────────────────────────────────────────────────────────────────┐
│                      Analysis Layer                              │
│  (Interprets data, detects issues, generates remediations)      │
└─────────────────────────────────────────────────────────────────┘
                              ▲
                              │ DiagnosticData interface
                              │
┌─────────────────────────────────────────────────────────────────┐
│                     Data Provider Layer                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │ Live SQL    │  │ File-based  │  │ AI-Generated            │  │
│  │ Server      │  │ (pre-gen)   │  │ Simulator               │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**Benefits**:

1. **Testability**: Test analysis logic with pre-generated data without needing a live SQL Server
2. **Reproducibility**: Save diagnostic data snapshots for regression testing
3. **AI Simulation**: Generate synthetic data for scenarios like "high wait states with deadlocks"
4. **Offline Analysis**: Analyze previously-collected data without server access

**Data Provider Interface**:
```go
type DiagnosticDataProvider interface {
    GetDMVData(query string) (*DMVResult, error)
    GetXEventData(session string, duration time.Duration) (*XEventStream, error)
    GetRingBufferData(bufferType string) (*RingBufferData, error)
    GetCatalogData(view string) (*CatalogResult, error)
    GetConnectionInfo() (*ConnectionInfo, error)
}
```

**Implementations**:
- `LiveSQLProvider` - queries real SQL Server
- `FileProvider` - reads from saved JSON/XML files
- `SimulatorProvider` - generates synthetic data based on scenario description

**Future: AI-Driven Test Data Generation**:
```bash
# Generate simulated data for testing analysis
sqlcmd troubleshoot connectivity --simulate "system with high wait states and a couple deadlocks"
sqlcmd troubleshoot connectivity --simulate "Kerberos auth failure due to encryption mismatch"
sqlcmd troubleshoot connectivity --simulate "certificate SAN mismatch with AG listener"
```

This enables training CSS engineers on the tool without needing real problem systems.

### Platform-Specific Implementations

**Windows**
- Use Windows APIs for Active Directory queries (LDAP)
- Access Windows Certificate Store for certificate validation
- Read Windows Event Logs via WMI or Event Log API
- Check local security policy for allowed encryption types
- Support named pipes connectivity testing
- BID tracing via registry key injection
- Network tracing via netsh (built-in)

**Linux**
- Use LDAP libraries for Active Directory queries (if joined to domain)
- Check Kerberos keytab configuration
- Read journald/syslog for log analysis
- Check krb5.conf for encryption type settings
- ODBC tracing via INI file settings
- Network tracing via tcpdump (built-in)
- Parse ODBC INI for DSN settings

**macOS**
- Similar to Linux for Kerberos configuration
- Read system.log and appropriate Apple logs
- Support Heimdal Kerberos configuration
- Network tracing via tcpdump

### Integration Points

- **go-mssqldb**: Use existing driver for connection testing
- **internal/localizer**: Use for localized error messages and output
- **cmd/modern/root**: Add new `troubleshoot` subcommand following existing patterns

### Data Collection Points (from SQL Check)

The support team's SQL Check tool collects:
- Computer name, Windows edition, Windows version
- Domain info, user context ("who am I")
- TLS versions enabled/disabled in registry
- Cipher suites and protocol order
- ODBC drivers and their capabilities
- OLEDB providers
- Encryption aliases
- Disabled loopback check setting
- SQL Server instance configuration (registry keys)
- Service account information and encryption types
- SPN registration status
- Certificate properties (thumbprint, SAN, CN, permissions)
- Force Encryption settings

### Known Issues to Detect

| Issue | Detection Method | Remediation |
|-------|-----------------|-------------|
| Diffie-Hellman zero-padding bug | Check cipher suite list | Disable older DH ciphers |
| RC4/AES encryption mismatch | Query msDS-SupportedEncryptionTypes | Set-ADUser command |
| Duplicate SPNs | Query AD for MSSQLSvc SPNs | setspn -D to remove |
| Missing listener in SAN | Compare connection name to cert SAN | Request new certificate |
| High connection timeout | Parse connection string | Investigate underlying issue |
| TNIR timeout confusion | Detect TNIR behavior | Explain segmentation |

### Phase 2 Enhancements (Future)

Based on CSS feedback, Phase 2 could include:
- Automated analysis/recommendations from collected data (AI-assisted)
- Integration with AI/Copilot for troubleshooting suggestions
- Connection pool issue detection (driver-level pooling diagnostics)
- Filter driver detection (BID shows call, network trace shows nothing)
- Advanced XEvent analysis and correlation
- Query Store integration for historical connection patterns
- Persistent monitoring mode for intermittent issues

---

## Open Questions

1. **Active Directory Access**: ~~How should the tool behave when it cannot query AD (non-domain joined, permissions, Linux without AD integration)?~~ **DECIDED**: Warn user but continue with available checks. Tool will detect AD accessibility at startup and display warning like "AD queries unavailable - some Kerberos diagnostics will be skipped" then proceed with non-AD checks.
2. **Remote vs Local**: ~~Should the tool support running diagnostics remotely (against a server the user can't log into) vs locally on the SQL Server machine?~~ **DECIDED**: Support both. When running remotely, clearly indicate which checks cannot be performed (e.g., "Server-side log analysis requires running on [servername] - skipping") and provide suggestion like "For complete diagnostics, run: sqlcmd troubleshoot connectivity --server localhost on the SQL Server machine".
3. **Azure SQL Support**: ~~What subset of diagnostics apply to Azure SQL Database/Managed Instance?~~ **DECIDED**: Do what we can with clear messaging on scenarios that won't work (e.g., "Windows Auth not supported on Azure SQL Database"). Additionally, include Azure-specific troubleshooting for:
   - **Azure SQL Database**: AAD/Entra auth validation, firewall rule checks, connection policy (Redirect vs Proxy), gateway connectivity
   - **SQL Managed Instance**: VNet connectivity, public endpoint config, AAD auth, redirect mode
   - **SQL Database in Fabric**: Entra auth, workspace permissions, endpoint format validation
   - **Synapse DW**: Dedicated vs serverless endpoints, AAD auth, firewall, pause/resume state detection
   - **Fabric DW**: Entra auth, workspace connectivity, endpoint format validation
4. **Credential Handling**: ~~How should the tool handle credentials for AD queries vs SQL Server connection?~~ **DECIDED**: The tool should walk the user through getting signed in with the proper credentials to test a scenario. Interactive prompts like "To test Windows Auth, sign in as the user experiencing the issue" or "To query AD for SPN information, you need domain credentials - enter username:" with clear explanation of why each credential is needed.
5. **Network Trace Permissions**: ~~netsh requires admin on Windows, tcpdump may require root on Linux - how to handle gracefully?~~ **DECIDED**: Prompt for runas/sudo elevation (e.g., "Network tracing requires admin privileges. Run with elevation? [Y/n]") but allow user to continue with best effort if they decline. Display message like "Continuing without network trace - some diagnostics will be limited."
6. **Trace File Size**: ~~How to prevent trace files from growing too large during retry tests?~~ **DECIDED**: User configurable options with auto-rotation and max file count as the default. Options: `--trace-max-size 50MB` (per file), `--trace-max-files 5` (rotation count), `--trace-mode circular|sequential`. Default: 50MB files, 5 file rotation, sequential mode.
7. **TDS Parsing**: ~~Should we include TDS protocol parsing like TDS View did? (Advanced, likely Phase 2)~~ **DECIDED**: Yes, include TDS protocol parsing. Will be implemented as Phase 2+ feature after core troubleshooting is complete. Parse TDS packets from network captures to show prelogin, login, SSPI token exchange, and error responses in human-readable format.

---

## References

- Meeting recordings with CSS support team (Oct 2025, Feb 2026)
- SQL Check PowerShell script
- SQL Trace utility
- SQL Network Analyzer (Malcolm's tool)
- BID Trace Analyzer
- Kerberos Configuration Manager
- TDS Parser/TDS View (legacy)
- Microsoft documentation on Kerberos encryption types
- Microsoft documentation on SQL Server TLS configuration
- netsh trace documentation
- tcpdump documentation
