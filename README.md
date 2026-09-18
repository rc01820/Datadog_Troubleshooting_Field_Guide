# Datadog Troubleshooting Field Guide

> **The 2:13 AM runbook.** Use this guide when the dashboard turns red and you need to determine whether the problem is the monitored system, the Datadog collector, the data pipeline, the query, or the notification path.
>
> **Audience:** Datadog administrators, platform engineers, SRE/NOC teams, network engineers, application teams, and developers.
>
> **Last reviewed:** September 17, 2026
>
> **Example convention:** Examples use generic Neomon Labs-style values such as `env:prod`, `app:abc`, `costcenter:noc`, `site:bos`, and `team:noc`.
>
> **Important:** Datadog features, metric names, integration versions, menu paths, and product availability vary by Datadog site, Agent version, integration version, entitlements, and organization configuration. Verify metric/tag names in Metrics Explorer or the appropriate Explorer before changing production monitoring.

---

## Table of Contents

1. [How to Use This Guide](#1-how-to-use-this-guide)
2. [Universal Triage Flow](#2-universal-triage-flow)
3. [First Five Minutes](#3-first-five-minutes)
4. [Agent Not Reporting](#4-agent-not-reporting)
5. [Host Missing](#5-host-missing)
6. [Metric Missing](#6-metric-missing)
7. [Integration Check Failing](#7-integration-check-failing)
8. [Remote Configuration Unavailable](#8-remote-configuration-unavailable)
9. [SNMP Device Unreachable](#9-snmp-device-unreachable)
10. [NDM Discovery Slow](#10-ndm-discovery-slow)
11. [Interface Missing](#11-interface-missing)
12. [Synthetic Test Failing](#12-synthetic-test-failing)
13. [Private Location Offline](#13-private-location-offline)
14. [Logs Missing](#14-logs-missing)
15. [APM Traces Missing](#15-apm-traces-missing)
16. [vSphere Metrics Incomplete](#16-vsphere-metrics-incomplete)
17. [Docker Containers Missing](#17-docker-containers-missing)
18. [Kubernetes Cluster Missing](#18-kubernetes-cluster-missing)
19. [Monitor Not Alerting](#19-monitor-not-alerting)
20. [Notification Not Delivered](#20-notification-not-delivered)
21. [Dashboard Shows No Data](#21-dashboard-shows-no-data)
22. [API Returns 400 / 401 / 403 / 429](#22-api-returns-400--401--403--429)
23. [Time / NTP Problems](#23-time--ntp-problems)
24. [Agent Flare and Escalation](#24-agent-flare-and-escalation)
25. [Fast Command Reference](#25-fast-command-reference)
26. [Troubleshooting Decision Trees](#26-troubleshooting-decision-trees)
27. [Escalation Evidence Template](#27-escalation-evidence-template)
28. [Official References](#28-official-references)

---

# 1. How to Use This Guide

Every symptom section follows the same model:

```text
Symptoms
   ↓
Checks
   ↓
Commands
   ↓
Likely Causes
   ↓
Resolution
   ↓
Validation
```

The goal is to move from **evidence to cause**, not from symptom to random configuration changes.

## 1.1 The most important troubleshooting question

Ask:

> **At which layer does the signal disappear?**

A Datadog alerting path usually looks like this:

```text
Monitored resource
      │
      ▼
Collector / Agent / Integration / Synthetic Worker
      │
      ▼
Network path to Datadog
      │
      ▼
Datadog intake
      │
      ▼
Metrics / Logs / APM / NDM / Synthetics
      │
      ▼
Query / Dashboard
      │
      ▼
Monitor
      │
      ▼
Notification Rule / Mention / Integration
      │
      ▼
Responder
```

If the metric never arrives, changing a monitor will not fix the problem.

If the monitor transitions correctly, changing the Agent will not fix a notification-routing problem.

## 1.2 Preserve evidence before changing configuration

Capture:

```text
Exact time of failure:
Timezone:
Affected host/device/test:
Datadog site:
Agent version:
Integration version:
Exact query:
Current tags:
Expected result:
Actual result:
Recent change:
Screenshot:
Relevant log lines:
```

For transient incidents, screenshots and exact timestamps are gold. Once the state recovers, some clues evaporate.

---

# 2. Universal Triage Flow

Use this order for almost every Datadog problem.

## Step 1: Confirm the underlying system

```text
Is the server actually alive?
Is the endpoint actually reachable?
Is the network device actually responding?
Is the application actually producing logs/traces?
```

Do not assume the dashboard is the source of truth until you verify independently.

## Step 2: Confirm the collector

Examples:

```text
Datadog Agent
Cluster Agent
SNMP check
vSphere check
Docker Agent
Synthetics Private Location
APM tracer
```

## Step 3: Confirm transport

Check:

```text
DNS
TCP/443
proxy
firewall
TLS inspection
system clock
Datadog site
```

## Step 4: Confirm telemetry exists in Datadog

Go to the most basic product surface:

```text
Metrics Explorer
Log Explorer / Live Tail
Trace Explorer
NDM Devices
Synthetic Test Results
Containers
Infrastructure Host List
```

## Step 5: Reproduce the query

Strip the query down.

Example:

```text
Original:
avg:system.cpu.user{env:prod AND app:abc AND site:bos} by {host}

Step 1:
avg:system.cpu.user{*}

Step 2:
avg:system.cpu.user{env:prod}

Step 3:
avg:system.cpu.user{env:prod AND app:abc}

Step 4:
add site:bos
```

The filter that makes the data disappear is usually your clue.

## Step 6: Check monitor/dashboard behavior

Only after confirming the data exists.

## Step 7: Check notification routing

Only after confirming the monitor transitioned.

---

# 3. First Five Minutes

When an incident starts, use this checklist before diving deep.

## 3.1 Agent-backed host

Linux:

```bash
sudo systemctl status datadog-agent
sudo datadog-agent status
sudo datadog-agent health
```

Windows PowerShell:

```powershell
Get-Service DatadogAgent

& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" status
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" health
```

## 3.2 Integration

Linux:

```bash
sudo -u dd-agent datadog-agent check <CHECK_NAME>
```

Windows:

```powershell
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" check <CHECK_NAME>
```

## 3.3 Network

Linux:

```bash
nslookup <target>
curl -vk https://<target>
ip route
```

Windows:

```powershell
Resolve-DnsName <target>
Test-NetConnection <target> -Port 443
```

## 3.4 Kubernetes

```bash
kubectl get pods -A
kubectl get daemonset -A
kubectl get deployment -A | grep -i datadog
kubectl get pods -A | grep -i datadog
```

## 3.5 Docker

```bash
docker ps
docker ps -a
docker logs <container>
docker inspect <container>
```

## 3.6 SNMP

First validate the network and credentials independently of the dashboard.

```text
Ping, if allowed
UDP/161 path
SNMP version
SNMP credentials
SNMP walk
```

---

# 4. Agent Not Reporting

## Symptoms

Typical symptoms:

```text
Host monitor alerts
Host disappears from active Infrastructure view
No new system metrics
datadog.agent.up stops reporting
Agent status last seen becomes stale
Integration data stops at the same time
```

You may also see:

```text
No Data
host not reporting
Agent service stopped
forwarder errors
API key errors
```

## Checks

Work in this order:

```text
1. Is the operating system alive?
2. Is the Datadog Agent service running?
3. Can the Agent read its configuration?
4. Is the API key correct?
5. Is the Datadog site correct?
6. Can the Agent reach Datadog over HTTPS?
7. Is a proxy required?
8. Is DNS working?
9. Is system time correct?
10. Is more than one Agent installed/running unexpectedly?
```

## Commands

### Linux

```bash
sudo systemctl status datadog-agent
sudo datadog-agent status
sudo datadog-agent health
sudo journalctl -u datadog-agent.service
```

Restart only after collecting useful evidence:

```bash
sudo systemctl restart datadog-agent
```

Agent configuration:

```text
/etc/datadog-agent/datadog.yaml
```

Agent logs are typically under:

```text
/var/log/datadog/
```

### Windows

```powershell
Get-Service DatadogAgent

& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" status
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" health
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" diagnose
```

Restart:

```powershell
Restart-Service DatadogAgent
```

or:

```powershell
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" restart-service
```

Agent log:

```text
C:\ProgramData\Datadog\logs\agent.log
```

### Network checks

Windows:

```powershell
Test-NetConnection <DATADOG_INTAKE_OR_SITE> -Port 443
Resolve-DnsName <DATADOG_INTAKE_OR_SITE>
```

Linux:

```bash
curl -vk https://<DATADOG_INTAKE_OR_SITE>
nslookup <DATADOG_INTAKE_OR_SITE>
```

## Likely Causes

```text
Agent service stopped
Agent crash
bad YAML/configuration
incorrect API key
incorrect site
proxy changed
firewall change
DNS failure
TLS interception/certificate problem
NTP/time drift
host resource exhaustion
disk full
Agent upgrade problem
duplicate/conflicting Agent installation
```

## Resolution

1. Correct the configuration or infrastructure issue.
2. Restart the Agent if the change requires it.
3. Re-run `status`.
4. Verify the Forwarder section has no persistent errors.
5. Verify system checks resume.
6. Verify expected integrations resume.

Do not use repeated restarts as diagnosis. A restart may erase the most useful transient state.

## Validation

Datadog:

```text
Infrastructure → Hosts
Metrics Explorer
Monitor status
```

Verify:

```text
new system metrics have current timestamps
host is active
datadog.agent.up is healthy where applicable
integration checks resume
monitor recovers
```

Example Metrics Explorer sanity query:

```text
avg:system.cpu.user{host:<hostname>}
```

---

# 5. Host Missing

## Symptoms

```text
Agent appears to run locally
Host cannot be found in Infrastructure
Host appears under unexpected name
Two hosts appear for one server
Cloud instance exists but expected Agent host does not
Host has stale data only
```

## Checks

```text
Agent hostname
hostname aliases
Datadog site
API key organization
host tags
recent host rename
cloud hostname behavior
container hostname behavior
duplicate Agent
time range
```

## Commands

Linux:

```bash
sudo datadog-agent hostname
sudo datadog-agent status
hostname
hostname -f
```

Windows:

```powershell
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" hostname
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" status

hostname
```

Search Datadog by:

```text
hostname
alias
IP-related tag if available
cloud instance ID
app tag
site tag
```

## Likely Causes

```text
Agent reporting a different hostname
hostname changed
FQDN vs short-name mismatch
cloud integration entity vs Agent entity confusion
wrong Datadog org/site
host filtered out of the current view
stale host aged out
duplicate hosts caused by hostname changes
container hostname detection problem
```

## Resolution

Standardize hostname strategy.

Do not change hostname configuration casually on established fleets because it can create a new Datadog host identity.

If a rename is required:

```text
1. Document old identity.
2. Change deliberately.
3. Verify expected new host.
4. Update monitors/dashboards using explicit host names.
5. Allow old identity to age out.
```

## Validation

```text
Infrastructure Host List shows expected host
current system metrics arrive
expected tags present
host monitor recognizes the host
no accidental duplicate active identity
```

---

# 6. Metric Missing

## Symptoms

```text
Metric no longer appears in Metrics Explorer
Dashboard widget is empty
Monitor reports No Data
Only some hosts/devices report the metric
Metric existed before an Agent/integration upgrade
```

## Checks

Ask four questions:

```text
Does the integration/check run?
Does the integration still emit this metric?
Does the resource generate the metric?
Does the query scope filter it out?
```

Start broad.

```text
<metric>{*}
```

Then narrow.

## Commands

Agent check:

Linux:

```bash
sudo -u dd-agent datadog-agent check <CHECK_NAME>
```

Windows:

```powershell
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" check <CHECK_NAME>
```

Agent status:

```bash
sudo datadog-agent status
```

or Windows:

```powershell
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" status
```

## Query Isolation

Suppose this returns nothing:

```text
avg:system.cpu.user{env:prod AND app:abc AND site:bos} by {host}
```

Test:

```text
avg:system.cpu.user{*}
```

Then:

```text
avg:system.cpu.user{env:prod}
```

Then:

```text
avg:system.cpu.user{env:prod AND app:abc}
```

Then add:

```text
site:bos
```

## Likely Causes

```text
integration check not running
metric renamed/removed in integration version
resource does not expose metric
wrong metric namespace
wrong tag
case mismatch in tag value
filter excludes data
metric delayed
metric sparse
monitor/dashboard time range too small
Agent or integration upgrade changed collection behavior
permissions prevent collection
```

## Resolution

Correct the first layer where data disappears.

Examples:

```text
Fix integration credentials
Correct integration YAML
Correct metric name
Correct filter/tag
Increase time range for sparse metric
Correct source permissions
Enable required optional metric collection
```

Do not invent a replacement metric with a similar name without verifying semantics and units.

## Validation

```text
Metric visible in Metrics Explorer
Expected groups visible
Current timestamp
Correct unit
Correct tags
Dashboard repopulates
Monitor evaluates
```

---

# 7. Integration Check Failing

## Symptoms

```text
Integration appears under Running Checks with ERROR
Last Successful Execution = Never
Metrics from one integration missing
Service check is critical
Agent itself is otherwise healthy
```

## Checks

```text
configuration file path
file name
YAML syntax
endpoint
credentials
permissions
DNS
port
TLS
integration version
resource API compatibility
timeout
proxy
```

## Commands

### Linux

Datadog documents running a check as the Agent user:

```bash
sudo -u dd-agent datadog-agent check <CHECK_NAME>
```

Include rate metrics if useful:

```bash
sudo -u dd-agent datadog-agent check <CHECK_NAME> --check-rate
```

Agent status:

```bash
sudo datadog-agent status
```

Systemd logs:

```bash
sudo journalctl -u datadog-agent.service
```

### Windows

```powershell
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" check <CHECK_NAME>
```

Examples:

```powershell
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" check snmp
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" check vsphere
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" check docker
```

## Likely Causes

```text
invalid YAML
configuration in wrong directory
configuration has wrong extension
credentials invalid
source API permission changed
target not reachable
TLS certificate problem
unsupported option for installed integration version
resource API changed
integration package/version mismatch
timeout
```

## Resolution

Follow this order:

```text
1. Validate YAML.
2. Validate network independently.
3. Validate credentials independently.
4. Compare config with installed integration docs/sample.
5. Run the check directly.
6. Restart Agent if configuration change requires it.
7. Re-run check.
```

A correctly configured integration should appear under **Running Checks** without persistent warnings/errors.

## Validation

Check output should show:

```text
Last Successful Execution: recent
Metric Samples: > 0 when expected
Service Checks: expected
Errors: none
```

Then verify the integration's metric/service check in Datadog.

---

# 8. Remote Configuration Unavailable

## Symptoms

```text
Remote Configuration shows disabled/unavailable
Fleet Automation cannot perform remote action
Agent shows remote management enabled but Fleet action fails
Remote flare/upgrade/configuration action unavailable
Datadog Installer not running
```

## Checks

Treat these as separate components:

```text
Organization/site supports desired Fleet feature?
Agent version sufficient?
Remote Configuration enabled?
API key allowed for Remote Configuration?
Agent can reach Remote Configuration services?
Datadog Installer required and running for the requested action?
RBAC permission sufficient?
```

Datadog's current Fleet documentation lists these minimum Agent versions for several Remote Configuration-based Fleet capabilities:

```text
Remote flare        7.47+
Agent upgrades      7.66+
Agent configuration 7.73+
```

Datadog recommends Agent `7.66+` as a general Fleet Remote Configuration baseline, but individual Fleet features can require newer versions.

**Important:** availability can differ by Datadog site. Do not assume a commercial-site Fleet feature exists in a Federal/Gov organization.

## Commands

### Agent status

Linux:

```bash
sudo datadog-agent status
```

Windows:

```powershell
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" status
```

### Windows services

```powershell
Get-Service *Datadog*
```

Inspect specific services:

```powershell
Get-Service DatadogAgent
Get-Service | Where-Object {$_.DisplayName -like "*Datadog*"}
```

### Search Agent log

PowerShell:

```powershell
Select-String `
  -Path "C:\ProgramData\Datadog\logs\agent.log" `
  -Pattern "remote|config|installer" `
  -CaseSensitive:$false
```

Linux:

```bash
grep -Ei 'remote|config|installer' /var/log/datadog/agent.log
```

## Likely Causes

```text
feature not supported on current Datadog site
Agent too old for requested Fleet function
Remote Configuration disabled in Agent
Remote Configuration disabled at organization level
API key configuration not enabled for required capability
outbound endpoint blocked
proxy not configured
Datadog Installer stopped/not installed
insufficient RBAC permission
```

## Resolution

1. Verify the feature is supported on your **exact Datadog site**.
2. Verify Agent version against the requested Fleet action.
3. Confirm Remote Configuration state.
4. Confirm the correct API key/org.
5. Confirm network/proxy access.
6. Confirm Installer status if the specific action depends on it.
7. Upgrade the Agent/Installer using normal change control if required.

Do not infer that `Remote Management Status: Enabled` means every Fleet component is operational.

## Validation

```text
Agent status shows expected Remote Configuration state
Fleet inventory sees current Agent
requested remote action is available
test action succeeds on one pilot Agent
no remote-config connection errors in Agent log
```

Use a pilot ring before broad remote changes:

```text
fleet_ring:pilot
```

---

# 9. SNMP Device Unreachable

## Symptoms

```text
Device absent from NDM
Device shows unreachable
snmp.can_check critical
SNMP check errors
Autodiscovery completes but device is not found
Last Successful Execution = Never
```

## Checks

Validate from the **Agent doing the polling**:

```text
Device IP
routing
UDP/161
SNMP version
community or SNMPv3 user
auth protocol
auth key
privacy protocol
privacy key
security level
context, if used
ACL on network device
source IP permitted by device
```

## Commands

### Agent status

Linux:

```bash
sudo datadog-agent status
```

Look for:

```text
snmp
Autodiscovery
monitoring IP
Last Successful Execution
Error
```

### Datadog Agent SNMP walk - Linux

SNMP v2 example:

```bash
sudo -u dd-agent datadog-agent snmp walk <IP_ADDRESS> -C <COMMUNITY_STRING>
```

SNMP v3 pattern:

```bash
sudo -u dd-agent datadog-agent snmp walk <IP_ADDRESS> \
  -A <AUTH_KEY> \
  -a <AUTH_PROTOCOL> \
  -X <PRIV_KEY> \
  -x <PRIV_PROTOCOL>
```

Add required user/version options according to your Agent CLI/version.

### Windows

Run from an elevated command shell.

```cmd
cd "C:\Program Files\Datadog\Datadog Agent\bin"
```

SNMP v2:

```cmd
"%ProgramFiles%\Datadog\Datadog Agent\bin\agent.exe" snmp walk -v 2 -C <COMMUNITY> <IP>:161
```

SNMP v3:

```cmd
"%ProgramFiles%\Datadog\Datadog Agent\bin\agent.exe" snmp walk -v 3 -u <USER> -a <AUTH_PROTOCOL> -A <AUTH_KEY> -x <PRIV_PROTOCOL> -X <PRIV_KEY> <IP>:161
```

### Independent SNMP client

If installed:

```bash
snmpwalk -v2c -c '<community>' <IP> 1.3.6.1.2.1.1
```

For v3 use the exact security parameters for the device.

## Likely Causes

```text
wrong credentials
SNMPv3 auth/privacy mismatch
ACL blocks Agent source address
device not routed from Agent
firewall blocks UDP/161
wrong port
wrong namespace/config
device only allows specific management station
credential rotated
device SNMP engine/problem
```

## Resolution

Fix connectivity/credentials first.

Do not attempt profile tuning until basic SNMP walk succeeds.

Recommended order:

```text
1. Reach device IP.
2. Confirm UDP/161 path.
3. Make SNMP walk succeed.
4. Run Datadog SNMP check.
5. Verify NDM device.
6. Verify profile/metrics.
```

## Validation

```text
Datadog Agent SNMP walk returns OIDs
snmp check has successful execution
device appears in NDM
reachability/service check becomes OK
device metadata populated
```

---

# 10. NDM Discovery Slow

## Symptoms

```text
Autodiscovery takes a long time
Thousands of IPs queued
Devices appear very slowly
Discovery appears stuck
High number of unreachable addresses
Agent resource use rises
SNMP polling begins long after Agent start
```

## Checks

Measure before increasing workers.

Collect:

```text
subnet size
number of scanned addresses
number reachable
number unreachable
SNMP timeout
retry count
number of credential profiles attempted
number of configured discovery workers
Agent CPU
Agent memory
scan completion time
```

## Commands

Agent status:

```bash
sudo datadog-agent status
```

Look for the **Autodiscovery** section.

It can show states such as:

```text
Subnet ... queued for scanning
Scanning subnet ...
Currently scanning IP ...
X IPs out of Y scanned
Subnet ... scanned
Found IPs ...
```

Windows:

```powershell
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" status
```

Test a sample device:

```bash
sudo -u dd-agent datadog-agent snmp walk <IP> ...
```

## Likely Causes

```text
very large CIDR ranges
most IPs do not contain SNMP devices
unreachable addresses wait for timeout
incorrect credentials attempted against many devices
network latency
packet filtering silently drops UDP instead of rejecting
too few discovery workers
Agent CPU saturation
multiple overlapping subnets
```

## Resolution

Optimize before simply increasing concurrency:

```text
1. Remove ranges that should not be scanned.
2. Split very large networks into deliberate discovery scopes.
3. Fix network ACLs.
4. Remove obsolete credential configurations.
5. Reduce unnecessary timeout/retry cost where supported and safe.
6. Measure Agent resource usage.
7. Increase workers only after proving worker capacity is the bottleneck.
```

A subnet containing thousands of mostly dead IPs can turn timeouts into a tiny hourglass factory. ⏳

## Validation

Compare before/after:

```text
total scan duration
devices discovered
reachable percentage
Agent CPU/memory
SNMP check duration
poll completion consistency
```

Also verify normal polling is not degraded by aggressive discovery concurrency.

---

# 11. Interface Missing

## Symptoms

```text
Device appears in NDM but an interface does not
Interface inventory incomplete
Interface metrics missing
Interface exists on device but cannot be found in Datadog
Interface aliases/descriptions missing
Interface status/utilization widget shows fewer ports than expected
```

## Checks

Determine which problem you have:

```text
A. Interface object is missing
B. Interface exists but metric is missing
C. Interface exists but expected tag is missing
D. Interface is filtered out
```

Check:

```text
IF-MIB support
ifTable
ifXTable
SNMP view/permissions
device profile
interface filters
interface configuration
interface type
Agent integration version
metric/tag actually emitted
```

## Commands

SNMP walk relevant MIB trees.

Basic system verification:

```bash
sudo -u dd-agent datadog-agent snmp walk <IP> ...
```

Run SNMP check:

```bash
sudo -u dd-agent datadog-agent check snmp
```

Windows:

```powershell
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" check snmp
```

Inspect Agent status:

```bash
sudo datadog-agent status
```

## Datadog Checks

In NDM/Metric Explorer inspect actual available tags. Common interface-oriented dimensions can include values such as:

```text
interface
interface_alias
interface_index
snmp_device
device_namespace
```

Do not assume all names are present on every metric.

## Likely Causes

```text
device does not expose expected IF-MIB entry
SNMP view excludes interface OIDs
profile does not map expected metric
interface excluded by filter
interface administratively removed
64-bit counter MIB not supported
metric/tag name differs from assumption
old Agent/integration version
```

## Resolution

1. Verify interface exists via SNMP from the polling Agent.
2. Verify Datadog profile/check receives the interface data.
3. Check interface filters.
4. Check profile mapping.
5. Check exact metric/tag emitted.
6. Upgrade Agent/integration if a known feature requires it.

## Validation

```text
interface appears in NDM
expected alias/index present
status metric present
traffic metrics present
monitor/dashboard query returns interface
```

---

# 12. Synthetic Test Failing

## Symptoms

```text
Synthetic API test fails
Browser test fails
one location fails while others pass
TIMEOUT
401/403
certificate error
element not found
response assertion failed
latency suddenly increases
```

## Checks

Split the failure into layers:

```text
DNS
TCP
TLS
HTTP
authentication
application response
assertion
browser step
location/network
```

Inspect the Synthetic result waterfall/timings.

## Commands

### HTTP

From a comparable source:

```bash
curl -vk https://<target>
```

Headers:

```bash
curl -vk -H 'Authorization: Bearer <token>' https://<target>
```

Windows:

```powershell
Invoke-WebRequest https://<target> -UseBasicParsing
Test-NetConnection <target> -Port 443
Resolve-DnsName <target>
```

### DNS

```bash
nslookup <target>
dig <target>
```

### TLS

```bash
openssl s_client -connect <target>:443 -servername <target>
```

## Likely Causes

```text
application actually down
DNS issue
TLS certificate chain problem
firewall
endpoint inaccessible from selected Synthetic location
authentication changed
secret expired
response format changed
assertion too brittle
browser selector changed
redirect change
latency increase
Private Location resource exhaustion
```

Datadog specifically documents `TIMEOUT` from a Private Location as commonly indicating that the Private Location cannot reach the target endpoint.

## Resolution

Match resolution to the layer:

```text
DNS        -> correct DNS/network
TLS        -> correct certificate chain/trust
401/403    -> credential/auth flow
timeout    -> network/path/endpoint
assertion  -> update only if application contract intentionally changed
browser    -> repair selector/workflow
location   -> fix or scale Private Location
```

Do not weaken assertions just to make a failing test green without confirming the expected application behavior.

## Validation

Run:

```text
Fast Test / Run Test
```

Verify:

```text
all expected locations pass
response assertions pass
latency normal
monitor recovers
no location-specific failures remain
```

---

# 13. Private Location Offline

## Symptoms

```text
Private Location status not reporting
Synthetic tests assigned to location stop running
default stopped-reporting monitor fires
worker container/service stopped
synthetics.pl.worker.running stops reporting
remaining slots exhausted
```

Datadog's Private Location monitoring includes a stopped-reporting monitor based on:

```text
synthetics.pl.worker.running
```

and an under-provisioned condition based on remaining worker slots.

## Checks

```text
worker running?
worker config valid?
outbound access to Datadog Synthetics intake?
DNS?
system clock?
CPU?
memory?
container OOM?
concurrency exhausted?
worker version?
```

## Commands

### Docker

```bash
docker ps
docker ps -a
docker logs <PRIVATE_LOCATION_CONTAINER>
docker inspect <PRIVATE_LOCATION_CONTAINER>
docker stats <PRIVATE_LOCATION_CONTAINER>
```

Check IP forwarding on Linux if logs suggest network forwarding problems:

```bash
sysctl net.ipv4.ip_forward
```

### Windows Private Location

Service:

```powershell
Get-Service -Name "Datadog Synthetics Private Location"
```

Restart:

```powershell
Restart-Service -Name "Datadog Synthetics Private Location"
```

Process:

```powershell
Get-Process synthetics-pl-worker -ErrorAction SilentlyContinue
```

Time:

```powershell
w32tm /query /status
```

## Likely Causes

```text
worker stopped
Docker host problem
configuration missing/corrupt
outbound Synthetics intake blocked
DNS failure
IPv4 forwarding disabled on container host
clock/NTP drift causing request-signature failure
OOM kill
CPU exhaustion
concurrency/remaining slots exhausted
old worker version
certificate interception
```

Datadog documents a `403` with an expired/not-yet-valid signature as potentially caused by clock skew on the worker host.

## Resolution

```text
Stopped worker     -> restart after determining cause
OOM                -> add memory / reduce load / add worker
High CPU           -> add CPU / horizontally scale
No slots           -> increase capacity/concurrency appropriately
Network            -> restore outbound access/DNS
Clock skew         -> fix NTP/time service
Old image          -> upgrade worker using Datadog installation instructions
```

Private Locations can be scaled:

```text
Vertically   more worker resources
Horizontally additional workers for the same location
```

## Validation

Datadog Private Locations should show healthy/reporting state.

Verify:

```text
synthetics.pl.worker.running reports
remaining slots healthy
test runs resume
Fast Test passes
worker logs show normal queue activity
```

---

# 14. Logs Missing

## Symptoms

```text
Expected log absent from Log Explorer
Live Tail empty
some services log, one does not
logs appear in Live Tail but not indexed search
trace exists but correlated log missing
```

## Checks

Follow the log pipeline:

```text
Application writes log
       ↓
Agent/container collector reads it
       ↓
Agent forwards it
       ↓
Datadog receives it
       ↓
Pipeline parses it
       ↓
Index / exclusion / retention decision
       ↓
User role/restriction query allows access
```

## Commands

Agent status:

Linux:

```bash
sudo datadog-agent status
```

Windows:

```powershell
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" status
```

Agent log:

Linux:

```bash
grep -Ei 'log|tail|error|warning' /var/log/datadog/agent.log
```

Windows:

```powershell
Select-String `
  -Path "C:\ProgramData\Datadog\logs\agent.log" `
  -Pattern "log|tail|error|warning" `
  -CaseSensitive:$false
```

Verify application writes data:

Linux:

```bash
tail -f /path/to/application.log
```

Docker:

```bash
docker logs <container>
```

Kubernetes:

```bash
kubectl logs <pod> -n <namespace>
```

## Likely Causes

```text
logs_enabled disabled
wrong file path
file permission
rotation behavior
container collection disabled
include/exclude filter
multiline rule issue
source/service incorrect
pipeline parsing issue
index filter
exclusion filter
retention
RBAC restriction query
wrong time range
wrong query
```

Datadog notes that if logs appear in Live Tail but not Log Explorer, indexes/exclusion behavior is an important place to investigate.

## Resolution

Find the first pipeline stage where the log disappears.

Examples:

```text
source file missing       -> application/logging issue
Agent not tailing         -> path/permission/config
Agent sees but not sends  -> forwarder/network
Live Tail only            -> index/exclusion/retention
Datadog has log but user does not -> RBAC/restriction query
query misses log          -> search syntax/attribute issue
```

## Validation

Use a unique test marker:

```text
DD_LOG_TEST_2026_09_17_001
```

Search:

```text
"DD_LOG_TEST_2026_09_17_001"
```

Verify:

```text
Live Tail
Log Explorer
correct service/source/env
expected parsed attributes
expected index
```

---

# 15. APM Traces Missing

## Symptoms

```text
Service absent from APM
application logs show trace connection errors
Agent shows no traces received
some services trace, one does not
traces stopped after deployment
localhost:8126 connection refused
```

## Checks

APM path:

```text
Application
   ↓
Datadog tracing SDK
   ↓
Agent trace receiver
   ↓
Datadog APM intake
   ↓
Trace Explorer / service
```

Check:

```text
tracer installed?
tracer enabled?
service/env/version?
Agent reachable?
correct host/port?
8126 accessible?
container networking correct?
sampling/retention?
SDK version compatible?
```

## Commands

Agent status:

Linux:

```bash
sudo datadog-agent status
```

Windows:

```powershell
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" status
```

Look for APM/Trace Agent information.

### Linux port

```bash
ss -lntp | grep 8126
```

or:

```bash
netstat -lntp | grep 8126
```

### Windows

```powershell
Get-NetTCPConnection -LocalPort 8126 -ErrorAction SilentlyContinue
Test-NetConnection 127.0.0.1 -Port 8126
```

### Application environment

Examples:

```text
DD_SERVICE
DD_ENV
DD_VERSION
DD_AGENT_HOST
DD_TRACE_AGENT_PORT
```

Enable tracer debug only for controlled troubleshooting because it can be verbose:

```text
DD_TRACE_DEBUG=true
```

Some tracers support:

```text
DD_TRACE_STARTUP_LOGS=true
```

## Container Warning

In a containerized deployment:

```text
localhost
```

usually refers to the application container itself, not the Agent container.

Use the correct Agent service/host strategy for your runtime.

## Agent 7.80+ Note

On Linux with newer Agent behavior, the trace Agent can use socket activation and may not appear running until trace data arrives. Therefore, a status line saying APM is not running before any trace submission is not automatically proof of a problem.

## Likely Causes

```text
tracer not loaded
wrong DD_AGENT_HOST
container points at localhost
port 8126 blocked
Agent APM disabled/misconfigured
SDK upgrade/instrumentation failure
application restarted without tracer
sampling/retention misunderstanding
service name changed
volume/cardinality limits
```

## Resolution

1. Confirm SDK loaded.
2. Confirm application can reach Agent trace receiver.
3. Confirm Agent receives spans.
4. Confirm Agent can reach Datadog.
5. Confirm service/env/version.
6. Confirm search time range and retention behavior.

## Validation

Agent should show traces received after generating test application traffic.

Search:

```text
service:<expected-service> env:<expected-env>
```

Verify:

```text
trace appears
service appears
resource appears
correct env/version
logs correlate where configured
```

---

# 16. vSphere Metrics Incomplete

## Symptoms

```text
vSphere integration is green but expected metrics missing
VMs visible but certain disk/CPU metrics absent
collection_level increased but metric still absent
per-core/per-disk metrics missing
property metrics missing
only some vCenter objects appear
```

## Checks

Datadog's vSphere integration collection depends on several independent controls:

```text
vCenter permissions
resource filtering
collection_level
per-resource vs per-instance metric
collect_per_instance_filters
collect_property_metrics
legacy vs current check
metric supported by vCenter/resource
```

Datadog documents that `collection_level` affects which metrics are collected, but it does not automatically enable all **per-instance** metrics.

For example, disk-specific metrics may require:

```yaml
collect_per_instance_filters:
  host:
    - 'disk\.totalLatency\.avg'
    - 'disk\.deviceReadLatency\.avg'
```

Property metrics require:

```yaml
collect_property_metrics: true
```

where applicable.

## Commands

Run the integration directly.

Linux:

```bash
sudo -u dd-agent datadog-agent check vsphere
```

Windows:

```powershell
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" check vsphere
```

Agent status:

```bash
sudo datadog-agent status
```

Search Agent log:

```bash
grep -i vsphere /var/log/datadog/agent.log
```

## Likely Causes

```text
metric outside selected collection level
metric is per-instance, not per-resource
collect_per_instance_filters not configured
property metrics disabled
vCenter read-only account lacks permission on object
"Propagate to children" missing
resource excluded by filter
legacy check behavior
metric not exposed by this vSphere object/version
check takes too long / API pressure
```

## Resolution

Recommended order:

```text
1. Verify vsphere.can_connect.
2. Verify vCenter user permissions.
3. Verify target object is discovered.
4. Check metric in Datadog vSphere integration docs.
5. Check collection_level.
6. Determine whether metric is per-resource or per-instance.
7. Add collect_per_instance_filters if required.
8. Enable property metrics only if needed.
9. Run the check directly.
```

Do not keep raising `collection_level` expecting it to override per-instance filtering.

## Validation

```text
check successful
target object visible
expected metric visible in Metrics Explorer
expected resource/instance tags present
dashboard query displays metric
```

---

# 17. Docker Containers Missing

## Symptoms

```text
Docker host visible but containers absent
Container Explorer empty
some containers missing
container logs missing
container metrics missing
Agent container running but no workload inventory
```

## Checks

```text
Agent running?
Docker socket accessible?
Process Agent/live container collection enabled?
container include/exclude rules?
Autodiscovery enabled?
Docker endpoint reachable?
container labels/tags?
```

## Commands

Docker host:

```bash
docker ps
docker ps -a
docker info
```

Agent container:

```bash
docker ps | grep -i datadog
docker logs <DATADOG_AGENT_CONTAINER>
docker exec -it <DATADOG_AGENT_CONTAINER> agent status
```

Inspect socket mount:

```bash
docker inspect <DATADOG_AGENT_CONTAINER>
```

Look for access to the Docker socket where required:

```text
/var/run/docker.sock
```

## Logs

If container logs are expected, verify log collection configuration. In container deployments, relevant settings can include:

```text
DD_LOGS_ENABLED
DD_LOGS_CONFIG_CONTAINER_COLLECT_ALL
```

depending on your chosen collection model.

## Likely Causes

```text
Docker socket not mounted
permission denied on Docker socket
Agent container unhealthy
live container collection disabled
Process Agent disabled
container excluded by discovery rules
wrong labels/Autodiscovery
logs collection disabled
Agent cannot reach workload endpoint
```

## Resolution

1. Verify Docker itself sees the container.
2. Verify Agent sees Docker.
3. Verify Agent status contains container/process components.
4. Verify include/exclude settings.
5. Verify required mounts/permissions.
6. Restart/redeploy Agent only after configuration correction.

## Validation

Datadog:

```text
Infrastructure → Containers
```

Verify:

```text
container appears
correct image/name
CPU/memory data current
expected tags
logs, if configured
```

---

# 18. Kubernetes Cluster Missing

## Symptoms

```text
Kubernetes cluster absent from Datadog
nodes appear but cluster/orchestrator view does not
Cluster Agent not connected
pods missing
orchestrator explorer disabled
cluster name missing
```

## Checks

```text
Node Agent DaemonSet
Cluster Agent Deployment
Agent pods running
Cluster Agent Service
RBAC
cluster name
Process Agent/live container collection
Orchestrator Explorer
Cluster Agent auth token
Kubelet connectivity
```

## Commands

### Inventory

```bash
kubectl get pods -A | grep -i datadog
kubectl get daemonset -A | grep -i datadog
kubectl get deployment -A | grep -i datadog
kubectl get service -A | grep -i datadog
```

### Logs

```bash
kubectl logs -n <namespace> <DATADOG_AGENT_POD>
kubectl logs -n <namespace> <DATADOG_CLUSTER_AGENT_POD>
```

### Agent status

```bash
kubectl exec -it -n <namespace> <DATADOG_AGENT_POD> -- agent status
```

### Cluster Agent

```bash
kubectl exec -it -n <namespace> <DATADOG_CLUSTER_AGENT_POD> -- agent status
```

Metadata map:

```bash
kubectl exec -it -n <namespace> <DATADOG_CLUSTER_AGENT_POD> -- agent metamap
```

### Cluster Agent connectivity indicators

Inspect node Agent environment:

```bash
kubectl exec -it -n <namespace> <DATADOG_AGENT_POD> -- env | grep DATADOG_CLUSTER_AGENT
```

## Cluster Name

A missing cluster name can disable some cluster/orchestrator functionality.

Datadog Operator conceptual example:

```yaml
spec:
  global:
    clusterName: prod-cluster
```

Helm conceptual example:

```yaml
datadog:
  clusterName: prod-cluster
```

## Likely Causes

```text
Cluster Agent not deployed
Cluster Agent service missing
Node Agents cannot resolve/reach Cluster Agent
auth token mismatch
RBAC incomplete
cluster name cannot be detected
Process Agent disabled
orchestrator explorer disabled
Kubelet TLS/connectivity issue
Agent pods crashlooping
```

## Resolution

Follow component order:

```text
Kubernetes resources
   ↓
Node Agent
   ↓
Cluster Agent
   ↓
Node Agent ↔ Cluster Agent connection
   ↓
Kubelet/API permissions
   ↓
Datadog intake
```

Do not troubleshoot a dashboard until `agent status` shows the Kubernetes components healthy.

## Validation

```text
cluster appears in Datadog
nodes current
pods current
deployments/services present
Cluster Agent connected
expected cluster_name tag
container/orchestrator views populate
```

---

# 19. Monitor Not Alerting

## Symptoms

```text
Metric crossed visible threshold but monitor stayed OK
Monitor query has No Data
one group alerts while another does not
monitor evaluation says skipped
No Data alert expected but never sent
cloud metric shows data after monitor evaluated
```

## Checks

Start with monitor history.

Check:

```text
underlying query
evaluation window
aggregation
grouping
threshold
evaluation delay
new group delay
require full window
No Data setting
group retention
downtime/mute
custom schedule
rollup
sparse metric
arithmetic NaN
```

## Commands / Queries

Copy the data query into the appropriate Explorer.

Metric example:

```text
avg:system.cpu.user{env:prod AND app:abc} by {host}
```

Graph over:

```text
2-3x the monitor evaluation window
```

If a 5-minute monitor behaves strangely, inspect 15 minutes or more.

## Sparse Data

Datadog documents that sparse metrics can cause evaluations to be skipped when no points are present in the evaluation window.

Consider:

```text
larger evaluation window
Do not require a full window
evaluation delay
default_zero() only when zero has correct semantic meaning
```

## Delayed Cloud Metrics

For AWS/crawler-based metrics, Datadog currently recommends an evaluation delay of at least:

```text
900 seconds / 15 minutes
```

for common delayed-metric cases.

## Rollups

A monitor rollup can produce No Data if a complete rollup bucket is not available.

If using an explicit rollup interval, evaluate whether the monitor truly needs it.

## No Data Notification

If you expect a No Data notification:

```text
If data is missing -> Show NO DATA and notify
```

Message must either apply generally or include:

```handlebars
{{#is_no_data}}
No Data: {{monitor.name}} has stopped reporting.
{{/is_no_data}}
```

## Likely Causes

```text
data did not exist in evaluation window
data delayed
require full window prevents evaluation
rollup misaligned with evaluation
new group delay
wrong tag scope
group missing required tag
arithmetic query returns NaN
threshold/operator misunderstood
monitor in downtime
No Data notify not enabled
conditional notification excludes No Data
```

## Resolution

Fix the evaluation semantics rather than lowering thresholds blindly.

Use:

```text
correct window
correct delay
correct grouping
correct No Data behavior
correct query
correct missing-data policy
```

## Validation

Use monitor history to verify:

```text
data points exist
evaluation occurs
threshold crossing creates expected state
group identity correct
recovery works
No Data behavior works if configured
```

---

# 20. Notification Not Delivered

## Symptoms

```text
Monitor shows Alert but no email/Slack/Teams/PagerDuty/webhook
one app routes correctly, another does not
Notification Rule seems ignored
recovery not delivered
No Data notification missing
```

## Checks

Separate monitor evaluation from delivery:

```text
1. Did monitor state transition?
2. Did notification message include recipient?
3. Did Notification Rule match?
4. Did integration accept delivery?
5. Did downstream service receive it?
```

## Monitor Notification Rule Checks

Notification Rules route based on the **monitor notification tagset**.

Example rule:

```text
app:abc AND env:prod
```

Verify the monitor itself has appropriate notification tags.

Do not confuse:

```text
Monitor metadata:
app:abc

Metric query scope:
avg:metric{app:abc}
```

The resource scope does not necessarily create monitor notification metadata.

Rule scope supports Boolean logic such as:

```text
AND
OR
NOT
```

Datadog documents full-key wildcard matching such as:

```text
env:*
```

Partial wildcard patterns are not generally equivalent in Notification Rule scope.

## Commands / API Checks

If using a webhook:

```bash
curl -vk https://<webhook-target>
```

Check receiving service logs.

If using API-driven Notification Rules, retrieve the rule using the appropriate Datadog API and inspect its scope/recipients.

## Direct Mentions

Datadog notifications using an `@notification` mention require correct formatting. In ordinary monitor messages, ensure the mention is separated appropriately from preceding text.

Example:

```text
Disk space is low @ops@example.com
```

## Likely Causes

```text
monitor did not transition
rule does not match notification tags
wrong AND/OR logic
monitor missing tag
recipient integration disconnected
invalid recipient
conditional template suppresses state
downtime suppresses expected notification
webhook endpoint unavailable
email filtered downstream
```

## Resolution

Build a controlled test:

```text
1. Create/test one monitor.
2. Apply exact monitor tags.
3. Force a safe state transition.
4. Confirm Notification Rule match.
5. Confirm recipient.
6. Confirm recovery.
```

Do not test bulk notification routing with hundreds of production monitors.

## Validation

Capture:

```text
monitor event
rule matched
delivery/integration event
receiver log/message
recovery notification
```

---

# 21. Dashboard Shows No Data

## Symptoms

```text
Widget says No Data
Metrics Explorer has data
one template variable selection empties widget
dashboard worked yesterday
table shows fewer groups than expected
monitor widget differs from metric widget
```

## Checks

Compare dashboard query with the data source.

Check:

```text
time range
template variable
saved view
query
tag syntax
grouping
formula
rollup
unit
widget source
permission/restriction
data delay
```

## Query Isolation

Example dashboard query:

```text
avg:system.cpu.user{$env AND $app} by {host}
```

Replace variables temporarily with literal values:

```text
avg:system.cpu.user{env:prod AND app:abc} by {host}
```

Then remove filters:

```text
avg:system.cpu.user{*} by {host}
```

## Template Variable Checks

If:

```text
$app = app:abc
```

verify the widget is using the correct representation for the source.

Typical forms include:

```text
$app
$app.value
```

These are not interchangeable in every context.

## Monitor Summary Widget

Monitor Summary search is not metric search.

Examples:

```text
tag:app:abc
```

means monitor metadata.

```text
scope:app:abc
```

means monitor query scope.

Using the wrong one can make a perfectly healthy widget look empty.

## Likely Causes

```text
template variable value has no matching data
tag exists on host but not metric
wrong query language for widget
time range too short
saved view overrides variable
rollup/formula produces NaN
group-by tag missing
RBAC restriction
wrong monitor-list search field
```

## Resolution

Rebuild from simplest working query:

```text
raw metric
+ one filter
+ second filter
+ grouping
+ formula
+ template variables
```

This is faster than staring at a 180-character query and negotiating with it.

## Validation

```text
same query works in source Explorer
dashboard widget matches source
template variable switches correctly
expected groups present
time range changes behave normally
```

---

# 22. API Returns 400 / 401 / 403 / 429

# 22.1 HTTP 400 - Bad Request

## Symptoms

```text
400 Bad Request
JSON/request validation error
invalid query
invalid parameter
malformed filter
```

## Checks

```text
endpoint
API version
HTTP method
JSON syntax
Content-Type
required attributes
query syntax
time format
URL encoding
enum values
```

## Commands

Verbose curl:

```bash
curl -v \
  -H "DD-API-KEY: $DD_API_KEY" \
  -H "DD-APPLICATION-KEY: $DD_APP_KEY" \
  "https://<DATADOG_API_SITE>/api/..."
```

For POST:

```bash
curl -v -X POST \
  -H "DD-API-KEY: $DD_API_KEY" \
  -H "DD-APPLICATION-KEY: $DD_APP_KEY" \
  -H "Content-Type: application/json" \
  -d @request.json \
  "https://<DATADOG_API_SITE>/api/..."
```

Validate JSON:

```bash
python -m json.tool request.json
```

## Likely Causes

```text
malformed JSON
unsupported parameter
wrong API version
query not URL encoded
invalid enum
wrong body schema
wrong timestamp units
```

## Resolution

Compare request against documentation for the **exact endpoint**.

Do not assume an API filter accepts a UI query just because the text looks similar.

## Validation

Expected response code depends on endpoint, commonly:

```text
200
201
202
204
```

---

# 22.2 HTTP 401 - Unauthorized

## Symptoms

```text
401 Unauthorized
API key rejected
application key rejected
```

## Checks

```text
correct API key?
correct application key?
correct headers?
correct Datadog site?
key active?
environment variable empty?
extra whitespace?
```

## Commands

Inspect environment safely without printing the full secret:

Bash:

```bash
echo ${#DD_API_KEY}
echo ${#DD_APP_KEY}
```

PowerShell:

```powershell
$env:DD_API_KEY.Length
$env:DD_APP_KEY.Length
```

Validate API key using Datadog's validation endpoint for the correct site.

Concept:

```text
GET /api/v1/validate
```

## Likely Causes

```text
wrong key
revoked key
wrong site
missing header
application key/API key from different organization
secret not loaded
new key not yet fully propagated
```

Datadog documents eventual consistency for key changes, so a newly created or changed key can briefly produce authentication errors.

## Resolution

```text
correct site
correct key pair
recreate/re-scope key only if needed
allow brief propagation time for new key
use retry with short exponential backoff for automation
```

## Validation

Validation endpoint succeeds and intended API call succeeds.

---

# 22.3 HTTP 403 - Forbidden

## Symptoms

```text
403 Forbidden
permission authorization checks failed
valid key but action denied
read works, write fails
Actions/Workflow API fails
```

## Checks

```text
application key scopes
owner's role permissions
resource-specific restrictions
required endpoint permission
Actions API access if relevant
site/product availability
```

Application keys cannot grant more authority than the effective permissions available to their owner.

## Commands

Check the application's configured scopes in Datadog.

For automation, identify the endpoint's documented permission.

Examples of permission concepts:

```text
monitor_write
monitor_config_policy_write
dashboard read/write permissions
org/app-key permissions
```

Use exact permission names from the target endpoint documentation.

## Likely Causes

```text
missing application-key scope
user/service account lacks permission
resource restricted
Actions API access not enabled
product not available in organization/site
```

## Resolution

Apply least privilege:

```text
grant only required scope/permission
use service account for shared automation
enable special API access only when required
```

Avoid solving every 403 by turning the automation identity into an administrator.

## Validation

Retry exact request and confirm only the intended action is now permitted.

---

# 22.4 HTTP 429 - Too Many Requests

## Symptoms

```text
429 Too Many Requests
automation works intermittently
bulk migration stalls
monitor/dashboard script gets throttled
```

## Checks

Inspect response headers:

```text
X-RateLimit-Limit
X-RateLimit-Period
X-RateLimit-Remaining
X-RateLimit-Reset
X-RateLimit-Name
```

Datadog rate limits vary by endpoint/bucket.

## Commands

Curl headers:

```bash
curl -i \
  -H "DD-API-KEY: $DD_API_KEY" \
  -H "DD-APPLICATION-KEY: $DD_APP_KEY" \
  "https://<DATADOG_API_SITE>/api/..."
```

## Likely Causes

```text
polling too frequently
parallel workers exceed bucket
pagination loop too aggressive
multiple automations share same limit bucket
retry loop has no backoff
```

## Resolution

Use:

```text
rate-limit-aware retry
X-RateLimit-Reset
exponential backoff
jitter
pagination
caching
batch endpoints where supported
lower polling frequency
```

Pseudo-code:

```python
if response.status_code == 429:
    sleep(reset_seconds)
    retry()
```

For mature automation, prefer:

```text
exponential backoff + jitter + maximum retry count
```

## Validation

Track:

```text
429 count
X-RateLimit-Remaining
request rate
automation completion time
```

Datadog provides API usage metrics for rate-limited APIs that can help identify consumers using the rate-limit budget.

---

# 23. Time / NTP Problems

Time errors can masquerade as unrelated Datadog failures.

## Symptoms

```text
Private Location 403 signature error
metrics appear late
traces outside expected time range
authentication signatures invalid
monitor evaluates unexpectedly
```

## Checks

Linux:

```bash
timedatectl
date -u
chronyc tracking
```

Windows:

```powershell
Get-Date
w32tm /query /status
w32tm /query /source
```

## Likely Causes

```text
NTP disabled
wrong time source
VM clock drift
firewall blocks NTP
domain time hierarchy issue
timezone confusion during incident review
```

## Resolution

Restore authoritative time synchronization according to operating-system/domain policy.

## Validation

```text
clock synchronized
Datadog telemetry timestamps current
Private Location authentication succeeds
monitors evaluate expected window
```

---

# 24. Agent Flare and Escalation

When local troubleshooting is no longer productive, collect a flare.

Datadog's flare gathers Agent configuration and logs into a troubleshooting archive and sanitizes categories of sensitive values before submission. Follow your organization's data-handling policy and review the archive if required.

## Linux

```bash
sudo datadog-agent flare
```

With a support case:

```bash
sudo datadog-agent flare <CASE_ID>
```

## Docker

```bash
docker exec -it <DATADOG_AGENT_CONTAINER> agent flare <CASE_ID>
```

## Kubernetes

```bash
kubectl exec -it <DATADOG_AGENT_POD> -- agent flare <CASE_ID>
```

## Windows

```powershell
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" flare
```

## Before Escalating

Capture:

```text
Agent status
direct integration check
exact error
timestamps
Agent version
integration version
Datadog site
configuration excerpt with secrets removed
network test
recent change
flare
```

A support case with "SNMP broken" creates a scavenger hunt.

A support case with a timestamped SNMP walk failure, Agent check output, source Agent IP, target device IP, Agent version, config snippet, and flare creates a diagnosis.

---

# 25. Fast Command Reference

## Agent - Linux

```bash
sudo systemctl status datadog-agent
sudo systemctl restart datadog-agent
sudo datadog-agent status
sudo datadog-agent health
sudo datadog-agent hostname
sudo datadog-agent flare
sudo -u dd-agent datadog-agent check <CHECK_NAME>
sudo journalctl -u datadog-agent.service
```

## Agent - Windows

```powershell
Get-Service DatadogAgent

& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" status
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" health
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" hostname
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" diagnose
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" flare
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" check <CHECK_NAME>
```

## Docker

```bash
docker ps
docker ps -a
docker logs <container>
docker inspect <container>
docker stats <container>
docker exec -it <datadog-agent> agent status
```

## Kubernetes

```bash
kubectl get pods -A
kubectl get daemonset -A
kubectl get deployment -A
kubectl get service -A

kubectl logs -n <namespace> <pod>
kubectl exec -it -n <namespace> <agent-pod> -- agent status
kubectl exec -it -n <namespace> <cluster-agent-pod> -- agent status
kubectl exec -it -n <namespace> <cluster-agent-pod> -- agent metamap
```

## SNMP

Linux:

```bash
sudo -u dd-agent datadog-agent check snmp
sudo -u dd-agent datadog-agent snmp walk <IP> ...
```

Windows:

```powershell
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" check snmp
```

## vSphere

Linux:

```bash
sudo -u dd-agent datadog-agent check vsphere
```

Windows:

```powershell
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" check vsphere
```

## Network

Linux:

```bash
curl -vk https://<target>
nslookup <target>
dig <target>
ss -lntp
ip route
```

Windows:

```powershell
Test-NetConnection <target> -Port 443
Resolve-DnsName <target>
Get-NetTCPConnection
```

---

# 26. Troubleshooting Decision Trees

## 26.1 "No Data"

```text
Dashboard / Monitor says No Data
           │
           ▼
Does Metrics Explorer show raw metric?
      ┌────┴────┐
      │         │
     NO        YES
      │         │
      ▼         ▼
Collector /   Query / filter /
integration   grouping / formula /
problem       template variable
      │         │
      ▼         ▼
Agent check   Remove filters
status        one at a time
```

## 26.2 "Host Down"

```text
Host monitor alert
      │
      ▼
Can you reach OS?
 ┌────┴────┐
 │         │
NO        YES
 │         │
 ▼         ▼
Infrastructure Agent service?
incident       │
         ┌─────┴─────┐
         │           │
        DOWN        RUNNING
         │           │
         ▼           ▼
 Start/fix      agent status
 service        forwarder/site/key
```

## 26.3 "SNMP Device Missing"

```text
Device missing
    │
    ▼
Agent Autodiscovery finished?
   ┌┴┐
  NO YES
  │   │
  ▼   ▼
Wait/ Check SNMP walk
scope     │
        ┌─┴─┐
        │   │
       FAIL OK
        │   │
        ▼   ▼
 network/  profile/
 auth      filter/
           Datadog metadata
```

## 26.4 "Monitor Alert, No Notification"

```text
Monitor status = Alert
       │
       ▼
State transition occurred?
       │
       ▼
Notification Rule/direct recipient?
       │
       ▼
Monitor notification tags match?
       │
       ▼
Recipient integration healthy?
       │
       ▼
Downstream receiver accepted?
```

## 26.5 "APM Missing"

```text
No traces
   │
   ▼
Tracer loaded?
  ┌┴┐
 NO YES
 │   │
 ▼   ▼
Fix  Can app reach Agent:8126?
SDK       ┌┴┐
         NO YES
         │   │
         ▼   ▼
       network Agent receiving spans?
               ┌┴┐
              NO YES
              │   │
              ▼   ▼
            Agent  intake/search/
            config service naming
```

---

# 27. Escalation Evidence Template

Copy this into a support ticket or incident record.

```text
# Datadog Troubleshooting Evidence

Issue:
Business impact:
First observed:
Timezone:
Still occurring: Yes / No

Datadog site:
Organization/environment:
Product:
  [ ] Agent
  [ ] Metrics
  [ ] Logs
  [ ] APM
  [ ] NDM
  [ ] Synthetics
  [ ] Private Location
  [ ] vSphere
  [ ] Docker
  [ ] Kubernetes
  [ ] Monitor
  [ ] Notification
  [ ] API

Affected object:
Hostname/device/test:
IP, if relevant:
Application:
Site/location:

Agent version:
Integration version:
Worker version:
Tracer version:

Exact query:
Expected result:
Actual result:

Recent changes:
Change timestamp:

Commands executed:
1.
2.
3.

Important command output:

Relevant Agent log lines:

Independent connectivity test:

Metric/log/trace timestamp of last known good data:

Screenshots attached:
  [ ] status
  [ ] query
  [ ] error
  [ ] timeline

Flare collected:
Support case ID:
```

---

# 28. Official References

Use these as the authoritative starting points because Datadog behavior evolves.

## Agent

- Agent Troubleshooting  
  https://docs.datadoghq.com/agent/troubleshooting/

- Agent Commands  
  https://docs.datadoghq.com/agent/configuration/agent-commands/

- Troubleshoot an Agent Check  
  https://docs.datadoghq.com/agent/troubleshooting/agent_check_status/

- Integration Troubleshooting  
  https://docs.datadoghq.com/agent/troubleshooting/integrations/

- Agent Flare  
  https://docs.datadoghq.com/agent/troubleshooting/send_a_flare/

- Windows Agent  
  https://docs.datadoghq.com/agent/supported_platforms/windows/

- Linux Agent  
  https://docs.datadoghq.com/agent/supported_platforms/linux/

## Fleet / Remote Configuration

- Remote Configuration for Fleet Automation  
  https://docs.datadoghq.com/agent/guide/setup_remote_config/

## NDM / SNMP

- NDM Troubleshooting  
  https://docs.datadoghq.com/network_monitoring/devices/troubleshooting/

## Synthetics

- Synthetic Monitoring Troubleshooting  
  https://docs.datadoghq.com/synthetics/troubleshooting/

- Private Locations  
  https://docs.datadoghq.com/synthetics/platform/private_locations/

- Private Location Monitoring  
  https://docs.datadoghq.com/synthetics/platform/private_locations/monitoring/

## Logs

- Logs Troubleshooting  
  https://docs.datadoghq.com/logs/troubleshooting/

## APM

- APM Troubleshooting  
  https://docs.datadoghq.com/tracing/troubleshooting/

- APM Connection Errors  
  https://docs.datadoghq.com/tracing/troubleshooting/connection_errors/

## Containers / Kubernetes

- Container Troubleshooting  
  https://docs.datadoghq.com/containers/troubleshooting/

- Cluster Agent Troubleshooting  
  https://docs.datadoghq.com/containers/troubleshooting/cluster-agent/

- Kubernetes Agent Configuration  
  https://docs.datadoghq.com/containers/kubernetes/configuration/

## vSphere

- vSphere Integration  
  https://docs.datadoghq.com/integrations/vsphere/

## Monitors

- Troubleshooting Monitor Alerts  
  https://docs.datadoghq.com/monitors/guide/troubleshooting-monitor-alerts/

- Troubleshooting No Data  
  https://docs.datadoghq.com/monitors/guide/troubleshooting-no-data/

- Notification Rules  
  https://docs.datadoghq.com/monitors/notify/notification_rules/

- Notifications  
  https://docs.datadoghq.com/monitors/notify/

## API

- API Rate Limits  
  https://docs.datadoghq.com/api/latest/rate-limits/

- API and Application Keys  
  https://docs.datadoghq.com/account_management/api-app-keys/

---

# Final Rule

When troubleshooting Datadog, identify the first broken link:

```text
Resource
→ Collector
→ Network
→ Intake
→ Data
→ Query
→ Monitor
→ Notification
```

Fix that link first.

Everything downstream is usually just reporting the consequences.
