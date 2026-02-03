# Research: SQL Server Connection Troubleshooting

**Feature**: 001-troubleshooting  
**Date**: 2026-02-02  
**Sources**: CSS Support Team meetings (Oct 2025, Feb 2026), SQLCheck tool analysis, SQL Network Analyzer

## Summary

This document captures the technical research and domain knowledge gathered from the CSS support team regarding SQL Server connection troubleshooting. The goal is to replicate the mental flow chart that experienced support engineers use when diagnosing connection issues.

Key insight from Shon Hauck: *"If we use SQLCMD to turn on BID tracing, enable network trace, do a connection test, and stop both - we could do it all in one step and collect both traces. That would help with troubleshooting significantly."*

---

## Current CSS Tools and Their Limitations

### SQL Check (PowerShell Script)
Primary data collection tool used by CSS. Problems:
- Unsigned PowerShell script - many customers can't run it
- Windows-only - very limited Linux support
- Requires manual analysis of output

**Data it collects:**
- Computer name, Windows edition/version
- Domain information
- User context (who am I)
- TLS versions enabled/disabled in registry
- Cipher suites and protocol order
- ODBC/OLEDB drivers and capabilities
- Encryption aliases
- Driver versions (major/minor)
- Disabled loopback check setting

### SQL Trace
More detailed tracing utility, also PowerShell-based with same limitations.

### SQL Network Analyzer (Malcolm's tool)
Parses network traces (ETL/PCAP) and:
- Breaks out individual TCP conversations
- Shows time spent in each connection phase
- Identifies longest-running/most expensive connections
- Shows commands run on existing connections
- Identifies retransmits and errors

### BID Trace Analyzer
Breaks out driver diagnostic logs by:
- Thread ID
- Process ID
- Creates separate files per conversation
- Filters noise from multi-threaded applications

### Kerberos Configuration Manager
- Only checks SPNs (FQDN SPNs)
- No concept of clustering, listeners, failovers
- Does NOT check encryption levels
- Limited to basic SPN validation

### TDS Parser/TDS View (Legacy)
- Parses network traffic to show TDS protocol details
- Shows packet details and TDS flag breakouts
- Last updated for TDS 7.2
- Used for advanced escalation troubleshooting

---

## BID Tracing (Driver Diagnostics)

### What is BID Tracing?
BID (Built-In Diagnostics) provides detailed driver-level logging showing exactly what the driver is doing during connection and operations.

### Windows BID Tracing
Requires:
1. Registry key to enable tracing
2. Provider GUID for the specific driver
3. Process recycle to inject the DLL

**Key insight**: If SQLCMD enables BID tracing programmatically before connecting, we don't need process recycle - we're starting fresh.

```
Registry Key: HKLM\SOFTWARE\Microsoft\BidInterface\Loader
```

### Linux BID Tracing
Much simpler - just settings in ODBC INI:
```ini
[ODBC]
Trace = Yes
TraceFile = /tmp/odbctrace.log
```

**Note**: On Linux it's not technically BID tracing, just file logging with same message format.

### What BID Traces Show
- Socket open requests and results
- DNS resolution details (A records returned, stored in linked list)
- TLS/SSL handshake details
- Login phase timing
- All connection parameters being used
- TNIR (Transparent Network IP Resolution) behavior
- Inner exception details not exposed to application

### Key BID Trace Scenarios

**Filter Driver Detection:**
> "I had to get a TTD trace and found Crowdstrike was zeroing out the address after we set it up. BID trace shows we call TCP open, we get an error back, but I never see it hit the wire at TCP layer." - Shon Hauck

If BID shows TCP open but network trace doesn't show it → filter driver is blocking.

**Connection Pool Issues:**
> "Unable to get connection from pool" - the outer exception doesn't give inner details. BID trace shows the actual inner exception (TCP timeout, etc.).

---

## Network Tracing

### Built-in OS Tools (No Installation Required!)

**Windows - Net SH:**
```cmd
# Start trace
netsh trace start capture=yes tracefile=c:\temp\nettrace.etl

# Stop trace  
netsh trace stop
```
Output: ETL file (can be opened with Wireshark or Netmon)

**Linux - tcpdump:**
```bash
# Start capture
tcpdump -i eth0 -w /tmp/capture.pcap port 1433

# Stop with Ctrl+C
```
Output: PCAP file

### Port Filtering
Can filter to only capture specific ports:
- UDP 1434 (SQL Browser)
- TCP 1433 (default instance)
- Named instance ports

### Correlation with BID Trace
Network trace + BID trace together prove:
- Driver is doing what it's supposed to
- Where packets are going
- Who's sending resets
- If packets never hit the wire (filter driver)

---

## Pre-Test Preparation (Critical!)

Before running diagnostics, flush cached data:

**Flush DNS Cache:**
```cmd
# Windows
ipconfig /flushdns

# Linux
systemd-resolve --flush-caches
# or
sudo service nscd restart
```

**Flush Kerberos Tickets:**
```cmd
# Windows
klist purge

# Linux
kdestroy
```

**Why this matters:**
> "A lot of engineers mess that up. They didn't flush DNS cache, didn't flush Kerberos tickets before running the trace. We need you to do that so we can see exactly what's happening - the TGT, the TGS, how the TGS was formatted." - Shon Hauck

---

## Connection Phase Analysis

### Connection Timeline
1. **TCP Socket Open** - Usually very fast
2. **Pre-login** - TDS negotiation
3. **SSL/TLS Handshake** - Client/Server Hello, cipher negotiation
4. **Login Phase** - Authentication, SQL Server assigns SPID

### Diagnosing Where Time Is Spent

| Slow Phase | Likely Cause |
|------------|--------------|
| Socket Open | Firewall, DNS, network issues |
| SSL/TLS Handshake | Cipher mismatch, certificate issues |
| Login Phase | **SQL Server performance issue** |

> "When I see socket connection and SSL/TLS handshake are really quick but we're spending all time in login phase, I start thinking we need to stop - it's not a client issue. We need to look at SQL Server performance side." - Shon Hauck

### TNIR (Transparent Network IP Resolution)
With TNIR enabled, connection timeout is segmented:
- Full login timeout divided into shorter segments
- Internal timeout value changes
- Can confuse users who set 15 second timeout but see 2 second internal timeout

---

## Connection String Red Flags

### Non-Default Settings That Indicate Problems

| Setting | Red Flag Value | Implication |
|---------|---------------|-------------|
| Connection Timeout | >60 seconds | Band-aid fix for underlying issue |
| Command Timeout | Very high | Performance problems |

> "When I see connection timeout set to 120 seconds, right there, that's a red flag. Why are you setting this to such a high value? It's a Band-Aid fix." - Shon Hauck

### Settings to Dump
- All connection string parameters
- DSN settings (from ODBC INI on Linux)
- What parameters were modified in code vs connection string

---

## Known Issues Database

### Diffie-Hellman Bug
**Issue**: If the dynamically generated key ended in a zero, it would truncate the key length.

**Symptoms**: Intermittent SSL/TLS failures

**Solution**: Disable older Diffie-Hellman ciphers

> "That actually generated thousands of calls to CSS because it was an intermittent failure - only if that key, when dynamically generated, happened to have a zero at the end." - Shon Hauck

### Fabric First Connection Failure
**Issue**: First connection attempt gets "TCP refused by remote host", retry succeeds.

**Symptoms**: Every first connection fails, second succeeds (connection resiliency masks it)

**Investigation**: Need network trace to see who's sending the reset.

---

## Retry/Loop Testing

For intermittent issues, need to:
- Connect N times
- Sleep X milliseconds between attempts
- Report success/failure for each
- Continue until failure or max attempts

Example: "Try 500 connection attempts with 500ms delay, tell me if all succeeded or what error occurred when it failed."

---

## Low-Hanging Fruit Scenarios (CSS Priority List)

Based on CSS team feedback, these are the most common issues that waste support time:

### Tier 1: Stop Calling Us For These
1. **SPN Registration** - Missing or incorrect SPNs
2. **Delegation Settings** - Kerberos delegation not configured
3. **Encryption Type Mismatch** - RC4 vs AES (most common Kerberos issue)
4. **Duplicate SPNs** - Same SPN on multiple accounts
5. **SPN on Wrong Account** - Registered on computer instead of service account

### Tier 2: Common But Slightly More Complex
1. **Certificate SAN Issues** - Listener/short name not in SAN
2. **Cipher Suite Mistyping** - Copy-paste errors in policy configuration
3. **TLS Version Mismatch** - Client/server TLS incompatibility
4. **Force Encryption with Bad Cert** - Encryption required but cert invalid

### Tier 3: Requires Deeper Analysis
1. **NTLM Performance** - 1-second DC timeout, need to switch to Kerberos
2. **Multi-path Network Issues** - TTL mismatch indicating asymmetric routing
3. **Filter Driver Interference** - Crowdstrike, etc. blocking connections
4. **SQL Server Performance** - Slow login phase (server-side issue)

---

## Network Trace Analysis Patterns

### SYN-ACK-RESET Pattern
> "If we see a SYN-ACK reset, we know that the server is rejecting connections either by a firewall or by a perf issue on the SQL Server." - James Peterson

**Detection**: In network trace, look for:
1. Client sends SYN
2. Server responds SYN-ACK
3. Immediate RESET

**Causes**:
- Firewall blocking
- SQL Server performance issue
- Max connections reached

### TTL Analysis for Multi-Pathing
> "On the one side we see TTL of 120 and the other side we see TTL of 116. So you send traffic on one direction and return on a different path." - James Peterson

**Detection**: Compare TTL values in request vs response packets. If significantly different, traffic is taking different routes.

**Common in**: AWS, Azure, complex enterprise networks

### NTLM Performance Issues
> "NTLM occurs on the SQL Server, not the client. We only have one second for the DC to make that connection." - James Peterson

**Detection**: 
- Need two-side trace (client AND server)
- Look for delays in accept_security_context
- NTLM queuing on SQL Server side

**Solution**: Switch to Kerberos authentication

---

## SSL/TLS Error Flow Chart

From James Peterson's troubleshooting approach:

```
SSL/TLS Connection Error
        │
        ▼
┌─────────────────────────────────────────┐
│ 1. Check TLS Settings                   │
│    - Registry enabled/disabled versions │
│    - Client vs Server TLS support       │
└─────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────┐
│ 2. Check Cipher Suites                  │
│    - Are there typos in policy?         │
│    - Common ciphers between client/srv? │
│    - Cipher suite order correct?        │
└─────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────┐
│ 3. Check Certificates                   │
│    - Is cert expired?                   │
│    - Is connection name in SAN/CN?      │
│    - Chain valid and trusted?           │
│    - Private key permissions?           │
└─────────────────────────────────────────┘
```

**"Target principal name is incorrect"**: Usually certificate chain error or incorrect cert settings, not always Kerberos.

---

## Listener Mode (Stretch Goal)

CSS team proposed a powerful diagnostic mode:

### Concept
> "What if SQLCMD had the ability to do what we did with the old port query where you'd run it on one side, run it on the other, and do testing in between without the full SQL endpoint?" - Shon Hauck

### Use Cases
1. **Production Testing**: Test connectivity without impacting running SQL Server
2. **Firewall Validation**: Prove network path works without SQL involvement
3. **Isolate SQL vs Network**: Prove if issue is SQL performance or network
4. **Bypass Process Recycle**: No need to restart apps for tracing

### Implementation Idea
```bash
# On SQL Server machine (or any machine)
sqlcmd listen --port 14330

# On client machine
sqlcmd troubleshoot --server targetmachine --port 14330 --listener-test

# This proves:
# - Network path works
# - Firewalls allow traffic
# - No intermediate devices blocking
# All without touching actual SQL Server
```

### Two-Side Testing
> "Have them reach out to each other and wherever the connection breaks, you found the middle of it." - David Levy

Run listener on both ends, have them try to connect bidirectionally to find where break occurs.

---

## Production Constraints

### Why Data Collection Is Hard
1. **Peak Time Issues**: Problems occur during peak, but can't collect data then
2. **Process Recycle**: BID trace on Windows requires app restart
3. **Maintenance Windows**: Often 24+ hour wait for safe testing window
4. **Disk Space**: Network traces can get huge, customers worry about space
5. **Intermittent Issues**: May need to run for days to catch

### SQLCMD Advantages
- Fresh process each time = no recycle needed for BID trace
- Command line = easy to script and schedule
- Single tool = no installation/configuration drama
- Can run during peak without impacting production apps

---

## Multi-Domain Knowledge Gap

CSS identified that other teams (networking, AD) don't understand SQL connection flow:

### What Networking Team Looks For
- Resets
- Retransmits
- Protocol errors
- High-level issues

### What They DON'T Understand
- Socket → Pre-login → Client/Server Hello → Post-login flow
- TDS protocol specifics
- Where time is spent in connection phases
- SQL-specific timeout behaviors (TNIR, connection resiliency)

**Implication**: sqlcmd troubleshooting must be self-contained; can't rely on other teams understanding the output.

---

### The "Cannot Generate SSPI Context" Error

According to the CSS team, this error fundamentally means: **"Your service account can't decrypt the ticket being sent to it."**

**Root Causes (in order of frequency):**

1. **Encryption Type Mismatch** (Most Common)
   - Service account has only RC4 enabled
   - Computer accounts or client machines only allow AES
   - GPOs are being deployed to remove RC4 from newly installed servers
   - Active Directory never automatically assigns AES to normal accounts

2. **SPN Issues**
   - SPN registered on wrong account
   - Duplicate SPNs across multiple accounts
   - Missing SPNs

3. **Certificate/Encryption Issues**
   - Listener name not in certificate SAN
   - Using short name but certificate only has FQDN
   - Force encryption enabled with certificate mismatch

4. **DNS Suffix Issues**
   - Wrong domain suffix in DNS lookup
   - TGS request assembled with incorrect domain
   - SPN lookup fails due to domain mismatch

### Encryption Type Analysis

**Where to Check:**

1. **Service Account (AD)**: `msDS-SupportedEncryptionTypes` attribute
   - Value `0x18` (24) = AES-128 + AES-256
   - Value `0x4` (4) = RC4 only
   - Value `0x1C` (28) = RC4 + AES-128 + AES-256

2. **Computer Account (AD)**: Same attribute, check for AES-only

3. **Local Machine Policy**: 
   - `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Kerberos\Parameters`
   - `SupportedEncryptionTypes` value

**Fix Command (PowerShell):**
```powershell
# Update service account to support AES encryption
Set-ADUser -Identity "<ServiceAccount>" -KerberosEncryptionType AES128, AES256, RC4

# Or for computer account
Set-ADComputer -Identity "<ComputerName>" -KerberosEncryptionType AES128, AES256
```

### SPN Validation

**Expected SPNs for SQL Server:**
- `MSSQLSvc/<FQDN>:1433`
- `MSSQLSvc/<FQDN>`
- `MSSQLSvc/<ShortName>:1433`
- `MSSQLSvc/<ShortName>`
- For named instances: `MSSQLSvc/<FQDN>:<InstanceName>`

**SPN Query Commands:**
```powershell
# Find all SPNs for a server
setspn -L <ServiceAccount>

# Search for duplicate SPNs
setspn -X

# Query SPNs from AD
Get-ADUser -Identity <Account> -Properties servicePrincipalName | Select-Object -ExpandProperty servicePrincipalName
```

**What SQLCheck Collects:**
- Domain service account properties
- Encryption types for service account
- Encryption types for computer accounts
- SPN registration and duplicates
- Whether SPN is on correct account

---

## Certificate/TLS Troubleshooting

### Certificate Requirements for SQL Server

1. **Subject Alternative Name (SAN)** must contain:
   - The name used in the connection string
   - If using short name with encryption, short name must be in SAN
   - For AG listeners, listener name must be in SAN
   - For FCIs, virtual network name must be in SAN

2. **Permissions**:
   - SQL Server service account must have READ permission on private key

3. **Certificate Properties**:
   - Must have Server Authentication EKU
   - Must not be expired
   - Must be in Local Machine certificate store

### What SQLCheck Collects:
- Certificate thumbprint from registry
- Certificate SAN entries
- Certificate CN (Common Name)
- Private key permissions
- Force Encryption setting from registry

### Validation Flow:
1. Get certificate thumbprint from SQL Server configuration
2. Retrieve certificate from store
3. Check if connection name is in SAN or CN
4. Verify expiration
5. Check private key permissions for service account

**Registry Locations:**
```
HKLM\SOFTWARE\Microsoft\Microsoft SQL Server\<Instance>\MSSQLServer\SuperSocketNetLib
  - Certificate (thumbprint)
  - ForceEncryption (0/1)
```

---

## Connectivity Troubleshooting

### Basic Connectivity Checks (like portqry)

1. **TCP Port Check**
   - Test connection to specified port
   - Default: 1433 for default instance
   - Named instances: Query SQL Browser on UDP 1434

2. **SQL Browser Service**
   - Resolves instance name to port
   - UDP port 1434
   - Returns instance information

3. **Named Pipes** (Windows only)
   - `\\<ServerName>\pipe\sql\query`
   - Named instance: `\\<ServerName>\pipe\MSSQL$<InstanceName>\sql\query`

### Network Trace Information

The CSS team mentioned they can extract from network traces:
- Kerberos TGS responses from DCs
- Actual encryption type being used/sent
- This is advanced and may be out of scope for v1

---

## Log Analysis

### SQL Server Error Log

**Location:**
- Default: `C:\Program Files\Microsoft SQL Server\MSSQL<version>.<instance>\MSSQL\Log\ERRORLOG`
- Up to 6 archived logs (ERRORLOG.1, ERRORLOG.2, etc.)

**Connection-Related Errors to Look For:**
- Login failed for user
- Cannot generate SSPI context
- Connection timeout
- Certificate errors
- TLS/SSL errors

### Windows Event Logs

**Relevant Logs:**
- Application: SQL Server events
- System: Network, Kerberos events
- Security: Authentication failures

**Event IDs:**
- 4771: Kerberos pre-authentication failed
- 4776: NTLM authentication
- 4625: Account failed to log on

### Linux/macOS Logs

**Linux:**
- `/var/log/syslog` or `journalctl`
- SQL Server logs in configured location
- Kerberos: `/var/log/krb5kdc.log`

**macOS:**
- `/var/log/system.log`
- Console app for unified logging

---

## Troubleshooting Flow Chart

Based on CSS team's mental model:

```
Connection Error
      │
      ▼
┌─────────────────────────────────────────┐
│ 1. Basic Connectivity                    │
│    - Can you ping the server?            │
│    - Is the port open?                   │
│    - Is SQL Server listening?            │
└─────────────────────────────────────────┘
      │ Passes
      ▼
┌─────────────────────────────────────────┐
│ 2. "Cannot Generate SSPI Context"?       │
│    YES → Go to Kerberos Checks           │
│    NO  → Continue to Certificate Checks  │
└─────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────┐
│ 3. Kerberos Checks (if SSPI error)       │
│    a. Check encryption types             │
│       - Service account msDS-Supported   │
│       - Computer account allowed types   │
│       - Local secpol allowed types       │
│    b. Check SPNs                         │
│       - Exist?                           │
│       - Duplicates?                      │
│       - On correct account?              │
│    c. If all pass, check certificate     │
└─────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────┐
│ 4. Certificate Checks (if force encrypt)│
│    - Is connection name in SAN?          │
│    - Using short name vs FQDN?           │
│    - Certificate expired?                │
│    - Service account has permissions?    │
└─────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────┐
│ 5. Generate Remediation Commands         │
│    - PowerShell for AD changes           │
│    - Certificate request guidance        │
│    - Configuration changes               │
└─────────────────────────────────────────┘
```

---

## Existing Tools

### Kerberos Configuration Manager

- **Limitations noted by CSS team:**
  - Only checks SPNs (FQDN SPNs)
  - No concept of clustering, listeners, failovers
  - Does NOT check encryption levels
  - Does not detect the most common issue

### SQLCheck

- CSS team's internal data collection tool
- Collects comprehensive configuration data
- Requires manual analysis of output
- We want to automate this analysis

---

## Platform Considerations

### Windows
- Full Active Directory integration available
- Certificate Store access
- Windows Event Logs
- Named Pipes support
- Registry access for SQL configuration

### Linux
- AD integration via SSSD/Realmd if domain-joined
- Kerberos via MIT or Heimdal (krb5.conf)
- Keytab file analysis
- journalctl for logs
- SQL Server configuration in mssql.conf

### macOS
- Similar to Linux for AD/Kerberos
- Keychain for certificates
- Console/system.log for logs
- Less common for SQL Server hosting

---

## Implementation Notes

1. Start with connectivity testing (P1) - works on all platforms
2. Kerberos diagnostics more complex on non-Windows
3. Certificate validation differs significantly by platform
4. Log analysis needs platform abstraction layer

## Next Steps

1. Define data model for troubleshooting results
2. Create API contracts for each diagnostic type
3. Implement platform abstraction for AD/Kerberos queries
4. Build CLI integration following go-sqlcmd patterns
