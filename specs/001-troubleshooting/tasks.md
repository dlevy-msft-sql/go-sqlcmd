# Tasks: SQL Server Connection Troubleshooting

**Input**: Design documents from `/specs/001-troubleshooting/`
**Prerequisites**: plan.md (required), spec.md (required), research.md, data-model.md

**Tests**: Include unit tests for each component. Integration tests where SQL Server access is available.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, etc.)
- Include exact file paths in descriptions

## Path Conventions

- CLI commands: `cmd/modern/root/`
- Internal packages: `internal/troubleshoot/`
- Tests: `*_test.go` alongside implementation files

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [ ] T001 [P] Create `internal/troubleshoot/doc.go` with package documentation
- [ ] T002 [P] Create `internal/troubleshoot/types.go` with all data model types from data-model.md
- [ ] T003 [P] Create `internal/troubleshoot/config.go` for TroubleshootConfig handling
- [ ] T004 [P] Create `internal/troubleshoot/result.go` for TroubleshootResult aggregation
- [ ] T005 [P] Create `internal/troubleshoot/remediation.go` for remediation generation helpers
- [ ] T006 Add troubleshoot constants to `internal/localizer/constants.go`
- [ ] T007 [P] Create `cmd/modern/root/troubleshoot.go` skeleton command using Cobra
- [ ] T008 [P] Create unit tests `internal/troubleshoot/types_test.go` for JSON serialization

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before user story implementation

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T009 Create `internal/troubleshoot/output/output.go` with OutputFormatter interface
- [ ] T010 [P] Create `internal/troubleshoot/output/text.go` for text output formatting
- [ ] T011 [P] Create `internal/troubleshoot/output/json.go` for JSON output formatting
- [ ] T012 Create shared error handling utilities in `internal/troubleshoot/errors.go`
- [ ] T013 Create platform detection utility in `internal/troubleshoot/platform.go`

**Checkpoint**: Foundation ready - user story implementation can now begin

---

## Phase 3: User Story 1 - Basic Connectivity (Priority: P1) 🎯 MVP

**Goal**: Test TCP/port connectivity to SQL Server, similar to portqry functionality

**Independent Test**: Run `sqlcmd troubleshoot --server <host>` and see pass/fail for connectivity

### Tests for User Story 1

- [ ] T014 [P] [US1] Unit test for DNS resolution in `internal/troubleshoot/connectivity/dns_test.go`
- [ ] T015 [P] [US1] Unit test for TCP port testing in `internal/troubleshoot/connectivity/tcp_test.go`
- [ ] T016 [P] [US1] Unit test for SQL Browser in `internal/troubleshoot/connectivity/sqlbrowser_test.go`
- [ ] T017 [US1] Integration test for connectivity in `internal/troubleshoot/connectivity/connectivity_test.go`

### Implementation for User Story 1

- [ ] T018 [P] [US1] Create `internal/troubleshoot/connectivity/doc.go` package documentation
- [ ] T019 [P] [US1] Implement DNS resolution in `internal/troubleshoot/connectivity/dns.go`
- [ ] T020 [P] [US1] Implement TCP port testing in `internal/troubleshoot/connectivity/tcp.go`
- [ ] T021 [US1] Implement SQL Browser protocol in `internal/troubleshoot/connectivity/sqlbrowser.go`
- [ ] T022 [US1] Implement named pipes testing in `internal/troubleshoot/connectivity/namedpipes_windows.go`
- [ ] T023 [US1] Create stub `internal/troubleshoot/connectivity/namedpipes_unix.go` for non-Windows
- [ ] T024 [US1] Implement main orchestrator `internal/troubleshoot/connectivity/connectivity.go`
- [ ] T025 [US1] Create connectivity subcommand `cmd/modern/root/troubleshoot/connectivity.go`
- [ ] T026 [US1] Wire up troubleshoot command in `cmd/modern/root/troubleshoot.go`
- [ ] T027 [US1] Add localized strings for connectivity messages

**Checkpoint**: `sqlcmd troubleshoot --server <host>` works for basic connectivity testing

---

## Phase 4: User Story 2 - Connection String Validation (Priority: P1)

**Goal**: Parse and validate connection strings, identifying specific issues

**Independent Test**: Run `sqlcmd troubleshoot --connection-string "<connstr>"` and see validation results

### Tests for User Story 2

- [ ] T028 [P] [US2] Unit test for connection string parsing in `internal/troubleshoot/connstring/parser_test.go`
- [ ] T029 [P] [US2] Unit test for component validation in `internal/troubleshoot/connstring/validator_test.go`

### Implementation for User Story 2

- [ ] T030 [P] [US2] Create `internal/troubleshoot/connstring/doc.go` package documentation
- [ ] T031 [US2] Implement connection string parser in `internal/troubleshoot/connstring/parser.go`
- [ ] T032 [US2] Implement component validator in `internal/troubleshoot/connstring/validator.go`
- [ ] T033 [US2] Implement error categorization in `internal/troubleshoot/connstring/errors.go`
- [ ] T034 [US2] Create connection string command handling in `cmd/modern/root/troubleshoot.go`
- [ ] T035 [US2] Add localized strings for connection string messages

**Checkpoint**: Connection string validation works with clear error messages

---

## Phase 5: User Story 3 - Kerberos/SSPI Diagnostics (Priority: P1)

**Goal**: Diagnose "Cannot generate SSPI context" errors with specific remediation

**Independent Test**: Run `sqlcmd troubleshoot --server <host> --kerberos` to see Kerberos analysis

### Tests for User Story 3

- [ ] T036 [P] [US3] Unit test for SPN validation in `internal/troubleshoot/kerberos/spn_test.go`
- [ ] T037 [P] [US3] Unit test for encryption type parsing in `internal/troubleshoot/kerberos/encryption_test.go`
- [ ] T038 [P] [US3] Unit test for remediation generation in `internal/troubleshoot/kerberos/remediation_test.go`
- [ ] T039 [US3] Integration test with mocked AD in `internal/troubleshoot/kerberos/kerberos_test.go`

### Implementation for User Story 3

- [ ] T040 [P] [US3] Create `internal/troubleshoot/kerberos/doc.go` package documentation
- [ ] T041 [P] [US3] Define AD query interface in `internal/troubleshoot/kerberos/ad.go`
- [ ] T042 [US3] Implement Windows AD queries in `internal/troubleshoot/kerberos/ad_windows.go`
- [ ] T043 [US3] Implement Linux AD queries (SSSD) in `internal/troubleshoot/kerberos/ad_linux.go`
- [ ] T044 [US3] Implement stub for macOS in `internal/troubleshoot/kerberos/ad_darwin.go`
- [ ] T045 [US3] Implement SPN validation logic in `internal/troubleshoot/kerberos/spn.go`
- [ ] T046 [US3] Implement encryption type analysis in `internal/troubleshoot/kerberos/encryption.go`
- [ ] T047 [US3] Implement ticket cache inspection in `internal/troubleshoot/kerberos/ticket.go`
- [ ] T048 [US3] Implement krb5.conf parsing in `internal/troubleshoot/kerberos/krb5_unix.go`
- [ ] T049 [US3] Implement main orchestrator in `internal/troubleshoot/kerberos/kerberos.go`
- [ ] T050 [US3] Create Kerberos subcommand in `cmd/modern/root/troubleshoot/kerberos.go`
- [ ] T051 [US3] Add localized strings for Kerberos messages and remediations

**Checkpoint**: Kerberos diagnostics work with encryption type mismatch detection and SPN validation

---

## Phase 6: User Story 4 - Certificate/TLS Validation (Priority: P2)

**Goal**: Validate SQL Server TLS certificate configuration

**Independent Test**: Run `sqlcmd troubleshoot --server <host> --certificate` to see certificate analysis

### Tests for User Story 4

- [ ] T052 [P] [US4] Unit test for SAN validation in `internal/troubleshoot/certificate/san_test.go`
- [ ] T053 [P] [US4] Unit test for certificate parsing in `internal/troubleshoot/certificate/parse_test.go`
- [ ] T054 [US4] Integration test in `internal/troubleshoot/certificate/certificate_test.go`

### Implementation for User Story 4

- [ ] T055 [P] [US4] Create `internal/troubleshoot/certificate/doc.go` package documentation
- [ ] T056 [P] [US4] Define certificate store interface in `internal/troubleshoot/certificate/store.go`
- [ ] T057 [US4] Implement Windows cert store in `internal/troubleshoot/certificate/store_windows.go`
- [ ] T058 [US4] Implement Linux cert handling in `internal/troubleshoot/certificate/store_linux.go`
- [ ] T059 [US4] Implement macOS Keychain in `internal/troubleshoot/certificate/store_darwin.go`
- [ ] T060 [US4] Implement SAN validation in `internal/troubleshoot/certificate/san.go`
- [ ] T061 [US4] Implement expiration/chain validation in `internal/troubleshoot/certificate/validation.go`
- [ ] T062 [US4] Implement permission checking in `internal/troubleshoot/certificate/permissions.go`
- [ ] T063 [US4] Implement main orchestrator in `internal/troubleshoot/certificate/certificate.go`
- [ ] T064 [US4] Create certificate subcommand in `cmd/modern/root/troubleshoot/certificate.go`
- [ ] T065 [US4] Add localized strings for certificate messages

**Checkpoint**: Certificate validation works with SAN checking and clear remediation

---

## Phase 7: User Story 5 - Log Analysis (Priority: P2)

**Goal**: Extract and categorize connection-related errors from logs

**Independent Test**: Run `sqlcmd troubleshoot --server <host> --logs` to see log analysis

### Tests for User Story 5

- [ ] T066 [P] [US5] Unit test for ERRORLOG parsing in `internal/troubleshoot/logs/errorlog_test.go`
- [ ] T067 [P] [US5] Unit test for log categorization in `internal/troubleshoot/logs/categorize_test.go`
- [ ] T068 [US5] Integration test in `internal/troubleshoot/logs/logs_test.go`

### Implementation for User Story 5

- [ ] T069 [P] [US5] Create `internal/troubleshoot/logs/doc.go` package documentation
- [ ] T070 [P] [US5] Define log source interface in `internal/troubleshoot/logs/source.go`
- [ ] T071 [US5] Implement ERRORLOG parser in `internal/troubleshoot/logs/errorlog.go`
- [ ] T072 [US5] Implement Windows Event Log in `internal/troubleshoot/logs/eventlog_windows.go`
- [ ] T073 [US5] Implement journald reader in `internal/troubleshoot/logs/journald_linux.go`
- [ ] T074 [US5] Implement macOS log reader in `internal/troubleshoot/logs/syslog_darwin.go`
- [ ] T075 [US5] Implement log categorization in `internal/troubleshoot/logs/categorize.go`
- [ ] T076 [US5] Implement main orchestrator in `internal/troubleshoot/logs/logs.go`
- [ ] T077 [US5] Create logs subcommand in `cmd/modern/root/troubleshoot/logs.go`
- [ ] T078 [US5] Add localized strings for log analysis messages

**Checkpoint**: Log analysis works with categorized error output

---

## Phase 8: User Story 6 - System Information Collection (Priority: P1)

**Goal**: Collect comprehensive system information similar to SQL Check

**Independent Test**: Run `sqlcmd troubleshoot --sysinfo` to see system report

### Tests for User Story 6

- [ ] T079 [P] [US6] Unit test for OS info collection in `internal/troubleshoot/sysinfo/os_test.go`
- [ ] T080 [P] [US6] Unit test for driver enumeration in `internal/troubleshoot/sysinfo/drivers_test.go`
- [ ] T081 [P] [US6] Unit test for TLS settings in `internal/troubleshoot/sysinfo/tls_test.go`

### Implementation for User Story 6

- [ ] T082 [P] [US6] Create `internal/troubleshoot/sysinfo/doc.go` package documentation
- [ ] T083 [US6] Implement OS info collection in `internal/troubleshoot/sysinfo/os.go`
- [ ] T084 [US6] Implement Windows-specific sysinfo in `internal/troubleshoot/sysinfo/os_windows.go`
- [ ] T085 [US6] Implement Linux-specific sysinfo in `internal/troubleshoot/sysinfo/os_linux.go`
- [ ] T086 [US6] Implement macOS-specific sysinfo in `internal/troubleshoot/sysinfo/os_darwin.go`
- [ ] T087 [US6] Implement ODBC driver enumeration in `internal/troubleshoot/sysinfo/drivers.go`
- [ ] T088 [US6] Implement TLS/cipher suite detection in `internal/troubleshoot/sysinfo/tls.go`
- [ ] T089 [US6] Implement user context ("who am I") in `internal/troubleshoot/sysinfo/identity.go`
- [ ] T090 [US6] Implement ODBC INI parsing (Linux) in `internal/troubleshoot/sysinfo/odbcini.go`
- [ ] T091 [US6] Create sysinfo subcommand in `cmd/modern/root/troubleshoot/sysinfo.go`
- [ ] T092 [US6] Add localized strings for sysinfo messages

**Checkpoint**: System information collection works like SQL Check

---

## Phase 9: User Story 7 - Driver Diagnostic Tracing (Priority: P1)

**Goal**: Enable BID tracing, run connection, collect trace - all in one command

**Independent Test**: Run `sqlcmd troubleshoot --server <host> --trace` to collect driver trace

### Tests for User Story 7

- [ ] T093 [P] [US7] Unit test for Windows BID trace setup in `internal/troubleshoot/trace/bid_test.go`
- [ ] T094 [P] [US7] Unit test for Linux ODBC trace in `internal/troubleshoot/trace/odbc_test.go`

### Implementation for User Story 7

- [ ] T095 [P] [US7] Create `internal/troubleshoot/trace/doc.go` package documentation
- [ ] T096 [US7] Define trace interface in `internal/troubleshoot/trace/trace.go`
- [ ] T097 [US7] Implement Windows BID tracing in `internal/troubleshoot/trace/bid_windows.go`
- [ ] T098 [US7] Implement Linux ODBC tracing in `internal/troubleshoot/trace/odbc_linux.go`
- [ ] T099 [US7] Implement trace file management in `internal/troubleshoot/trace/files.go`
- [ ] T100 [US7] Create trace subcommand in `cmd/modern/root/troubleshoot/trace.go`
- [ ] T101 [US7] Add localized strings for trace messages

**Checkpoint**: Driver diagnostic tracing works with automatic enable/collect/stop

---

## Phase 10: User Story 8 - Network Trace Collection (Priority: P2)

**Goal**: Capture network traces using built-in OS tools (netsh/tcpdump)

**Independent Test**: Run `sqlcmd troubleshoot --server <host> --nettrace` to capture network

### Tests for User Story 8

- [ ] T102 [P] [US8] Unit test for netsh wrapper in `internal/troubleshoot/nettrace/netsh_test.go`
- [ ] T103 [P] [US8] Unit test for tcpdump wrapper in `internal/troubleshoot/nettrace/tcpdump_test.go`

### Implementation for User Story 8

- [ ] T104 [P] [US8] Create `internal/troubleshoot/nettrace/doc.go` package documentation
- [ ] T105 [US8] Define network trace interface in `internal/troubleshoot/nettrace/nettrace.go`
- [ ] T106 [US8] Implement Windows netsh wrapper in `internal/troubleshoot/nettrace/netsh_windows.go`
- [ ] T107 [US8] Implement Linux tcpdump wrapper in `internal/troubleshoot/nettrace/tcpdump_linux.go`
- [ ] T108 [US8] Implement macOS tcpdump wrapper in `internal/troubleshoot/nettrace/tcpdump_darwin.go`
- [ ] T109 [US8] Implement port filtering in `internal/troubleshoot/nettrace/filter.go`
- [ ] T110 [US8] Create nettrace subcommand in `cmd/modern/root/troubleshoot/nettrace.go`
- [ ] T111 [US8] Add localized strings for network trace messages

**Checkpoint**: Network tracing works using built-in OS tools

---

## Phase 11: User Story 9 - Retry/Stress Testing (Priority: P2)

**Goal**: Run multiple connection attempts for intermittent issue diagnosis

**Independent Test**: Run `sqlcmd troubleshoot --server <host> --retry 100 --delay 500`

### Tests for User Story 9

- [ ] T112 [P] [US9] Unit test for retry logic in `internal/troubleshoot/retry/retry_test.go`
- [ ] T113 [P] [US9] Unit test for result aggregation in `internal/troubleshoot/retry/results_test.go`

### Implementation for User Story 9

- [ ] T114 [P] [US9] Create `internal/troubleshoot/retry/doc.go` package documentation
- [ ] T115 [US9] Implement retry orchestrator in `internal/troubleshoot/retry/retry.go`
- [ ] T116 [US9] Implement result aggregation in `internal/troubleshoot/retry/results.go`
- [ ] T117 [US9] Implement stop-on-failure mode in `internal/troubleshoot/retry/stopper.go`
- [ ] T118 [US9] Create retry flags in `cmd/modern/root/troubleshoot.go`
- [ ] T119 [US9] Add localized strings for retry messages

**Checkpoint**: Retry testing works with configurable attempts and delays

---

## Phase 12: User Story 10 - Comprehensive Report Package (Priority: P2)

**Goal**: Generate complete diagnostic package (ZIP) with all collected data

**Independent Test**: Run `sqlcmd troubleshoot --server <host> --full-report --output report.zip`

### Tests for User Story 10

- [ ] T120 [P] [US10] Unit test for ZIP packaging in `internal/troubleshoot/report/package_test.go`
- [ ] T121 [P] [US10] Unit test for redaction in `internal/troubleshoot/report/redact_test.go`
- [ ] T122 [US10] End-to-end test in `cmd/modern/root/troubleshoot_test.go`

### Implementation for User Story 10

- [ ] T123 [P] [US10] Create `internal/troubleshoot/report/doc.go` package documentation
- [ ] T124 [US10] Implement ZIP/CAB packaging in `internal/troubleshoot/report/package.go`
- [ ] T125 [US10] Implement redaction utility in `internal/troubleshoot/report/redact.go`
- [ ] T126 [US10] Implement upload instructions in `internal/troubleshoot/report/instructions.go`
- [ ] T127 [US10] Implement full report orchestration in `internal/troubleshoot/report/report.go`
- [ ] T128 [US10] Create report subcommand in `cmd/modern/root/troubleshoot/report.go`
- [ ] T129 [US10] Add localized strings for report generation

### Sensitive Data Filtering (Air-Gapped/Secure Environments)

- [ ] T129a [P] [US10] Unit test for sensitive data filtering in `internal/troubleshoot/report/filter_test.go`
- [ ] T129b [US10] Implement `--no-sensitive` flag to exclude sensitive data collection
- [ ] T129c [US10] Implement `--redact-patterns` flag for custom regex redaction patterns
- [ ] T129d [US10] Implement `--redact-hostnames` flag to anonymize server/usernames
- [ ] T129e [US10] Implement `--redact-config` flag to load redaction rules from file
- [ ] T129f [US10] Implement `--preview` flag to review collected data before packaging
- [ ] T129g [US10] Implement sensitive data filter in `internal/troubleshoot/report/filter.go`
- [ ] T129h [US10] Add summary output of excluded/redacted data types
- [ ] T129i [US10] Add localized strings for sensitive data filtering messages

**Checkpoint**: Full report packages generate with all diagnostic data

---

## Phase 13: User Story 11 - Connection Timing Analysis (Priority: P1)

**Goal**: Show exactly where time is spent during connection phases

**Independent Test**: Run `sqlcmd troubleshoot --server <host> --timing`

### Tests for User Story 11

- [ ] T130 [P] [US11] Unit test for phase timing in `internal/troubleshoot/timing/phases_test.go`
- [ ] T131 [P] [US11] Unit test for bottleneck detection in `internal/troubleshoot/timing/bottleneck_test.go`

### Implementation for User Story 11

- [ ] T132 [P] [US11] Create `internal/troubleshoot/timing/doc.go` package documentation
- [ ] T133 [US11] Implement connection phase instrumentation in `internal/troubleshoot/timing/phases.go`
- [ ] T134 [US11] Implement bottleneck detection in `internal/troubleshoot/timing/bottleneck.go`
- [ ] T135 [US11] Implement TNIR detection in `internal/troubleshoot/timing/tnir.go`
- [ ] T136 [US11] Create timing subcommand in `cmd/modern/root/troubleshoot/timing.go`
- [ ] T137 [US11] Add localized strings for timing messages

**Checkpoint**: Connection timing analysis identifies bottleneck phase

---

## Phase 14: User Story 12 - Listener Mode (Priority: P3 - Stretch Goal)

**Goal**: Test network connectivity without SQL Server involvement

**Independent Test**: Run `sqlcmd listen --port 14330` on target, test from client

### Tests for User Story 12

- [ ] T138 [P] [US12] Unit test for TCP listener in `internal/troubleshoot/listener/listener_test.go`
- [ ] T139 [P] [US12] Unit test for bidirectional test in `internal/troubleshoot/listener/bidir_test.go`

### Implementation for User Story 12

- [ ] T140 [P] [US12] Create `internal/troubleshoot/listener/doc.go` package documentation
- [ ] T141 [US12] Implement TCP listener in `internal/troubleshoot/listener/listener.go`
- [ ] T142 [US12] Implement listener-test client in `internal/troubleshoot/listener/client.go`
- [ ] T143 [US12] Implement bidirectional testing in `internal/troubleshoot/listener/bidir.go`
- [ ] T144 [US12] Create listen command in `cmd/modern/root/listen.go`
- [ ] T145 [US12] Add localized strings for listener messages

**Checkpoint**: Listener mode works for production-safe network testing

---

## Phase 15: User Story 13 - Delegation Settings Validation (Priority: P2)

**Goal**: Validate Kerberos delegation configuration for double-hop scenarios

**Independent Test**: Run `sqlcmd troubleshoot --server <host> --kerberos --check-delegation`

### Tests for User Story 13

- [ ] T146 [P] [US13] Unit test for delegation parsing in `internal/troubleshoot/kerberos/delegation_test.go`

### Implementation for User Story 13

- [ ] T147 [US13] Implement delegation type detection in `internal/troubleshoot/kerberos/delegation.go`
- [ ] T148 [US13] Add delegation to AD queries in `internal/troubleshoot/kerberos/ad.go`
- [ ] T149 [US13] Add delegation flags to Kerberos subcommand
- [ ] T150 [US13] Add localized strings for delegation messages

**Checkpoint**: Delegation validation identifies unconstrained/constrained/resource-based delegation

---

## Phase 16: Cross-Cutting - Graceful Degradation & Azure Support

**Goal**: Implement graceful degradation, guided credentials, and Azure platform support (Decisions 1-5 from Open Questions)

**Independent Test**: Run tool in various degraded scenarios and verify appropriate warnings/continuations

### Tests for Graceful Degradation

- [ ] T164 [P] Unit test for AD detection in `internal/troubleshoot/platform/ad_test.go`
- [ ] T165 [P] Unit test for remote vs local detection in `internal/troubleshoot/platform/location_test.go`
- [ ] T166 [P] Unit test for privilege detection in `internal/troubleshoot/platform/privileges_test.go`
- [ ] T167 [P] Unit test for Azure endpoint detection in `internal/troubleshoot/azure/detect_test.go`

### Implementation for Graceful Degradation

- [ ] T168 [P] Create `internal/troubleshoot/platform/doc.go` package documentation
- [ ] T169 Implement AD accessibility detection in `internal/troubleshoot/platform/ad.go`
- [ ] T170 Implement remote vs local detection in `internal/troubleshoot/platform/location.go`
- [ ] T171 Implement privilege detection in `internal/troubleshoot/platform/privileges.go`
- [ ] T172 Implement elevation prompts in `internal/troubleshoot/platform/elevate_windows.go`
- [ ] T173 Implement elevation prompts in `internal/troubleshoot/platform/elevate_unix.go`
- [ ] T174 Implement guided credential flow in `internal/troubleshoot/auth/guide.go`
- [ ] T175 Add degradation warnings to all diagnostic commands

### Implementation for Azure Platform Support

- [ ] T176 [P] Create `internal/troubleshoot/azure/doc.go` package documentation
- [ ] T177 Implement Azure endpoint detection in `internal/troubleshoot/azure/detect.go`
- [ ] T178 Implement Azure SQL Database diagnostics in `internal/troubleshoot/azure/sqldb.go`
- [ ] T179 Implement Azure SQL MI diagnostics in `internal/troubleshoot/azure/mi.go`
- [ ] T180 Implement Fabric SQL diagnostics in `internal/troubleshoot/azure/fabric.go`
- [ ] T181 Implement Synapse DW diagnostics in `internal/troubleshoot/azure/synapse.go`
- [ ] T182 Implement Entra ID auth validation in `internal/troubleshoot/azure/entra.go`
- [ ] T183 Add localized strings for degradation and Azure messages

**Checkpoint**: Tool gracefully handles missing AD, remote execution, missing elevation, and Azure platforms

---

## Phase 17: Cross-Cutting - Trace File Management

**Goal**: Implement configurable trace file rotation and size limits (Decision 6 from Open Questions)

**Independent Test**: Run `sqlcmd troubleshoot --trace --trace-max-size 10MB --trace-max-files 3` and verify rotation

### Tests for Trace File Management

- [ ] T184 [P] Unit test for file rotation in `internal/troubleshoot/trace/rotation_test.go`
- [ ] T185 [P] Unit test for size limits in `internal/troubleshoot/trace/limits_test.go`

### Implementation for Trace File Management

- [ ] T186 Implement file rotation in `internal/troubleshoot/trace/rotation.go`
- [ ] T187 Implement size monitoring in `internal/troubleshoot/trace/limits.go`
- [ ] T188 Add trace management flags to troubleshoot command
- [ ] T189 Add localized strings for trace management messages

**Checkpoint**: Trace files auto-rotate with configurable size and count limits

---

## Phase 18: User Story 14 - TDS Protocol Parsing (Priority: P3 - Phase 2)

**Goal**: Parse TDS protocol from network captures for human-readable analysis

**Independent Test**: Run `sqlcmd troubleshoot --parse-tds capture.pcap` to see TDS handshake analysis

### Tests for User Story 14

- [ ] T190 [P] [US14] Unit test for TDS prelogin parsing in `internal/troubleshoot/tds/prelogin_test.go`
- [ ] T191 [P] [US14] Unit test for TDS login7 parsing in `internal/troubleshoot/tds/login7_test.go`
- [ ] T192 [P] [US14] Unit test for SSPI token parsing in `internal/troubleshoot/tds/sspi_test.go`
- [ ] T193 [P] [US14] Unit test for pcap/etl reading in `internal/troubleshoot/tds/capture_test.go`

### Implementation for User Story 14

- [ ] T194 [P] [US14] Create `internal/troubleshoot/tds/doc.go` package documentation
- [ ] T195 [US14] Implement TDS packet framing in `internal/troubleshoot/tds/frame.go`
- [ ] T196 [US14] Implement prelogin parsing in `internal/troubleshoot/tds/prelogin.go`
- [ ] T197 [US14] Implement login7 parsing in `internal/troubleshoot/tds/login7.go`
- [ ] T198 [US14] Implement SSPI token parsing in `internal/troubleshoot/tds/sspi.go`
- [ ] T199 [US14] Implement error/info message parsing in `internal/troubleshoot/tds/messages.go`
- [ ] T200 [US14] Implement pcap file reading in `internal/troubleshoot/tds/pcap.go`
- [ ] T201 [US14] Implement etl file reading in `internal/troubleshoot/tds/etl_windows.go`
- [ ] T202 [US14] Create parse-tds subcommand in `cmd/modern/root/troubleshoot/tds.go`
- [ ] T203 [US14] Add localized strings for TDS parsing messages

**Checkpoint**: TDS parser shows human-readable handshake analysis from network captures

---

## Phase 19: Polish & Cross-Cutting Concerns

**Goal**: Final integration, documentation, and quality improvements

### Tests

- [ ] T204 [P] Integration tests with live SQL Server (optional, CI-configured)
- [ ] T205 [P] Performance tests for long-running operations
- [ ] T206 [P] Accessibility review for terminal output

### Final Tasks

- [ ] T207 Documentation updates in README.md for troubleshoot command
- [ ] T208 Add troubleshoot command examples to help text
- [ ] T209 Create troubleshooting tutorial/examples
- [ ] T210 Run `go generate` to update localization catalogs
- [ ] T211 Code cleanup and linting fixes
- [ ] T212 Performance optimization for large log files
- [ ] T213 Security review for credential handling
- [ ] T214 Add verbose/debug output mode
- [ ] T215 Localization review for all new strings
- [ ] T216 Run full test suite validation

**Checkpoint**: Feature complete, documented, and ready for release

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion
- **User Stories (Phase 3-15)**: All depend on Foundational phase completion
  - US1 (Connectivity) and US2 (Connection String) can proceed in parallel
  - US3 (Kerberos), US4 (Certificate), US5 (Logs) can proceed in parallel after US1
  - US6 (Sysinfo), US7 (BID Trace), US11 (Timing) are P1 and can run in parallel
  - US8 (Network Trace), US9 (Retry), US10 (Report), US13 (Delegation) are P2
  - US12 (Listener Mode) is P3 stretch goal
- **Cross-Cutting (Phase 16-17)**: Can be integrated alongside or after user stories
- **TDS Parsing (Phase 18)**: Phase 2 feature, depends on network trace infrastructure
- **Polish (Phase 19)**: Depends on all user stories being complete

### User Story Priority Order

**Priority 1 (P1) - Core Functionality:**
- **User Story 1**: Connectivity Diagnostics
- **User Story 2**: Connection String Parsing
- **User Story 3**: Kerberos/SSPI Diagnostics
- **User Story 6**: System Information Collection
- **User Story 7**: Driver Diagnostic Tracing (BID)
- **User Story 11**: Connection Timing Analysis

**Priority 2 (P2) - Enhanced Functionality:**
- **User Story 4**: Certificate Validation
- **User Story 5**: Log Analysis
- **User Story 8**: Network Trace Collection
- **User Story 9**: Retry/Stress Testing
- **User Story 10**: Comprehensive Report Package
- **User Story 13**: Delegation Settings Validation

**Priority 3 (P3) - Stretch Goals:**
- **User Story 12**: Listener Mode
- **User Story 14**: TDS Protocol Parsing (Phase 2)

### Parallel Opportunities

Within each user story, tasks marked [P] can run in parallel:

**Phase 1**: T001-T005, T007-T008 can all run in parallel
**Phase 3**: T014-T016, T018-T020 can run in parallel
**Phase 5**: T036-T038, T040-T041 can run in parallel
**Phase 8-15**: Test tasks within each phase can run in parallel

---

## Implementation Strategy

### Recommended MVP Approach

1. Complete Phase 1-2 (foundation)
2. Implement US1 fully (connectivity) - provides immediate value
3. Implement US3 (Kerberos) - addresses most common support issue
4. Implement US6 (Sysinfo) - SQL Check equivalent
5. Implement US7 (BID Trace) - diagnostic tracing
6. Implement US11 (Timing) - connection phase analysis
7. Implement US2 (connection string) - ties everything together
8. Then US4, US5, US8-10, US13 for full feature set
9. Phase 16-17 (graceful degradation, Azure support, trace management) can be woven in
10. US12 (Listener) as stretch goal if time permits
11. US14 (TDS Parsing) as Phase 2 stretch goal

### Platform Priority

1. Windows first (most common SQL Server environment)
2. Linux second (Azure VMs, containers)
3. macOS last (development scenarios)

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Avoid: vague tasks, same file conflicts, cross-story dependencies that break independence
- Follow existing go-sqlcmd patterns for CLI commands
- All user-facing strings MUST use internal/localizer package
- Total Tasks: T001-T216 + T129a-T129i across 19 phases (225 tasks)
- User Stories: 14 total (US1-US14)
