# Datadog Troubleshooting Field Guide

> **The 2:13 AM runbook.** Use this guide when the dashboard turns red and you need to determine whether the problem is the monitored system, the Datadog collector, the data pipeline, the query, or the notification path.
>
> **Audience:** Datadog administrators, platform engineers, SRE/NOC teams, network engineers, application teams, and developers.
>
> **Last reviewed:** September 20, 2026
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
25. [Custom Metrics and DogStatsD Not Arriving](#25-custom-metrics-and-dogstatsd-not-arriving)
26. [Cloud Integration Metrics Missing (AWS / Azure)](#26-cloud-integration-metrics-missing-aws--azure)
27. [Secret Backend Resolution Failures](#27-secret-backend-enc-resolution-failures)
28. [Tag Problems and Tag Fragmentation](#28-tag-problems-and-tag-fragmentation)
29. [Data Is Present but Wrong](#29-data-is-present-but-wrong)
30. [Downtimes, Mutes, and Suppressed Alerts](#30-downtimes-mutes-and-suppressed-alerts)
31. [Network Path and Endpoint Reference](#31-network-path-and-endpoint-reference)
32. [Windows-Specific Notes](#32-windows-specific-notes)
33. [Migration-Era Pitfalls](#33-migration-era-pitfalls)
34. [Fast Command Reference](#34-fast-command-reference)
35. [Troubleshooting Decision Trees](#35-troubleshooting-decision-trees)
36. [Escalation Evidence Template](#36-escalation-evidence-template)
37. [Official References](#37-official-references)

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

## 1.3 Classify the blast radius first

Before opening a single configuration file, establish scope. Scope changes both the likely cause and who needs to be involved.

```text
One metric on one host          -> check/integration on that host
All metrics on one host         -> Agent, host, or network path for that host
One integration everywhere      -> integration version, credentials, or upstream API
All hosts at one site/subnet    -> network, proxy, firewall, or TLS inspection change
All hosts everywhere            -> API key, org, site, or Datadog-side incident
One monitor only                -> monitor configuration or query
All notifications               -> routing, integration, or downstream receiver
```

A useful early question: **did anything still arrive during the outage window?**

```text
Metrics Explorer: avg:datadog.agent.running{*} by {host}
Event Explorer:   sources:datadog "agent"
Infrastructure:   sort Host List by "Last reported"
```

If a hundred hosts stopped at the same second, stop troubleshooting the Agent and start looking at the shared dependency: proxy, firewall rule, certificate, DNS, key rotation, or a site-wide change.

Also check Datadog's own status page before spending an hour proving that your Agent is healthy:

```text
https://status.datadoghq.com
```

Select the correct site (US1, US3, US5, EU1, AP1, US1-FED). A degraded intake or delayed metric pipeline on the Datadog side produces symptoms identical to a local collection failure.

## 1.4 Changes that quietly break monitoring

Most "nothing changed" incidents involve one of these:

```text
API or application key rotated, revoked, or re-scoped
Proxy or egress policy updated
TLS inspection enabled on a new subnet or firewall rule set
Certificate bundle updated on the host
DNS forwarders changed
Hostname, domain membership, or FQDN behavior changed
Agent or integration upgraded (metric renames, deprecations)
Golden image rebuilt without the Agent or with a stale configuration
SNMP credentials rotated on the network side only
Monitor or dashboard edited by another team member
Tag policy or tag value casing changed upstream
Cloud IAM role or integration permissions narrowed
Time source or NTP hierarchy changed
```

When someone says nothing changed, ask specifically about the list above. "Nothing changed" and "nothing changed that I personally did" are different statements.

## 1.5 What to check *before* declaring a Datadog problem

```text
Is the underlying resource actually healthy?
Did the data ever exist, or is this a first-time configuration that never worked?
Is this a display problem (time range, template variable, saved view)?
Is the user's role allowed to see this data?
Is the expectation itself correct?
```

The last one matters more than it sounds. A large share of "Datadog is broken" tickets are requests for a metric that the source system does not expose.

---

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

Each Datadog product uses its own intake hostname, so partial failures are normal and informative:

```text
Metrics arrive, logs do not      -> logs intake or logs configuration
Metrics arrive, traces do not    -> APM intake or trace Agent
Everything stops at once         -> shared transport (proxy/DNS/cert/key)
```

See section 31 for the endpoint and port reference used when writing firewall or proxy exceptions.

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

## 3.2 Self-diagnostics

Before reading configuration files by hand, let the Agent report on itself.

Linux:

```bash
sudo datadog-agent diagnose
sudo datadog-agent configcheck
```

Windows:

```powershell
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" diagnose
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" configcheck
```

`diagnose` runs built-in connectivity and configuration suites. `configcheck` prints the configuration the Agent **actually loaded**, including anything resolved through Autodiscovery, which is frequently different from what is sitting on disk.

## 3.3 Integration

Linux:

```bash
sudo -u dd-agent datadog-agent check <CHECK_NAME>
```

Windows:

```powershell
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" check <CHECK_NAME>
```

## 3.4 Network

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

## 3.5 Kubernetes

```bash
kubectl get pods -A
kubectl get daemonset -A
kubectl get deployment -A | grep -i datadog
kubectl get pods -A | grep -i datadog
```

## 3.6 Docker

```bash
docker ps
docker ps -a
docker logs <container>
docker inspect <container>
```

## 3.7 SNMP

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

### Deeper Agent diagnostics

Linux:

```bash
sudo datadog-agent diagnose
sudo datadog-agent configcheck
sudo datadog-agent config
sudo datadog-agent tagger-list
sudo datadog-agent version
sudo datadog-agent status -j > /tmp/agent-status.json
```

Windows:

```powershell
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" diagnose
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" configcheck
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" config
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" tagger-list
```

What each one answers:

```text
diagnose      problems the Agent can detect about itself (connectivity, config, permissions)
configcheck   which check configurations were actually loaded and from where
config        the effective runtime configuration, including defaults you never set
tagger-list   the tags the Agent will attach to what it collects on this host
status -j     machine-readable status, useful for scripted fleet health checks
```

Raise log verbosity temporarily without editing files or restarting (supported on recent Agent 7 versions):

```bash
sudo datadog-agent config set log_level debug
sudo datadog-agent config get log_level
sudo datadog-agent config set log_level info
```

This is a runtime override and does not survive a restart. Set it back when you are finished; debug logging is extremely verbose and can fill a disk on a busy host.

## How to Read `agent status`

Most Agent incidents are solved by reading this output properly rather than by restarting the service.

Sections worth reading, in order:

```text
Agent        version, hostname, clock offset, config file path
Forwarder    transaction counts, retry queue, API key validity
Collector    Running Checks, Last Successful Execution, Errors, Warnings
Autodiscovery container/SNMP discovery state
DogStatsD    packets received, packet errors
APM Agent    trace receiver state
Logs Agent   tailers, bytes sent, endpoint state
Aggregator   flush counts and flush errors
```

Red flags and what they mean:

```text
API Key ending with ...: API Key invalid   -> wrong key, wrong org, or wrong site
Transactions dropped: increasing           -> intake unreachable or persistently rejecting
Retry queue size: growing, never draining  -> transport blocked (proxy/firewall/TLS)
Clock offset: large                        -> NTP problem (see section 23)
Last Successful Execution: Never           -> the check has never worked; configuration issue
Errors: persistent                         -> integration problem, not a transport problem
Warnings: repeated                         -> permissions, deprecation, or partial collection
```

The most useful distinction in this output:

```text
Forwarder unhealthy, checks healthy  -> transport/credential problem
Forwarder healthy, one check failing -> that integration only
Forwarder healthy, all checks failing-> host-level problem (resources, permissions, clock)
```

Those are three different incidents with three different owners.

## Proxy, TLS, and Endpoint Checks

Agent proxy configuration lives in `datadog.yaml`:

```yaml
proxy:
  https: "http://user:password@proxy.example.com:3128"
  http: "http://user:password@proxy.example.com:3128"
  no_proxy:
    - 169.254.169.254
    - localhost
    - 127.0.0.1
```

Environment equivalents:

```text
DD_PROXY_HTTPS
DD_PROXY_HTTP
DD_PROXY_NO_PROXY
```

Notes that cause real outages:

```text
Agent proxy settings do not automatically apply to every integration or subprocess.
Cloud metadata endpoints usually belong in no_proxy.
A proxy that terminates TLS changes the certificate the Agent sees.
Proxy credentials embedded in configuration expire like any other secret.
```

TLS interception is the single most common silent breaker in enterprise networks. Typical Agent log evidence:

```text
x509: certificate signed by unknown authority
tls: failed to verify certificate
certificate is valid for <proxy vendor>, not <datadog endpoint>
```

Verify what the host actually receives:

```bash
openssl s_client -connect api.datadoghq.com:443 -servername api.datadoghq.com </dev/null 2>/dev/null \
  | openssl x509 -noout -issuer -subject -dates
```

If the issuer is your security appliance, traffic is being inspected. The correct fix is to trust the inspecting CA in the host trust store, or to exempt the Datadog domains from inspection. Disabling certificate validation (`skip_ssl_validation`) may prove the diagnosis in a lab, but it is not a production remedy.

Windows equivalent check:

```powershell
Test-NetConnection api.datadoghq.com -Port 443
[Net.ServicePointManager]::SecurityProtocol
```

## Which Host-Down Signal to Trust

```text
datadog.agent.up          service check; suitable for host/Agent down detection
Host monitor type         purpose-built native monitor
datadog.agent.running     metric; not a reliable down detector on its own
```

When an Agent dies, its metrics do not become `0` — they become **absent**. A monitor written as "alert when the value equals 0" therefore never fires, because there is no value to evaluate. Down detection must rely on service-check status or explicit No Data behavior, not on a numeric threshold.

Test this deliberately once, in a controlled way, before trusting host-down alerting across a fleet.

## Is More Than One Agent Running?

Duplicate or leftover installations produce confusing, intermittent symptoms.

Linux:

```bash
ps -ef | grep -i [d]atadog
systemctl list-units | grep -i datadog
ls -l /etc/datadog-agent/ /opt/datadog-agent/ 2>/dev/null
```

Windows:

```powershell
Get-Service *Datadog*
Get-Process | Where-Object {$_.Name -like "*agent*"}
Get-ChildItem "C:\Program Files\Datadog" -ErrorAction SilentlyContinue
```

Expected Agent 7 services on Windows:

```text
datadogagent            core Agent
datadog-trace-agent     APM
datadog-process-agent   live processes/containers
datadog-system-probe    network/system probe, when enabled
```

Two Agents on one host, or an Agent plus a legacy collector, can send conflicting host metadata and produce duplicate or flapping hosts.

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

## How the Agent Chooses a Hostname

The Agent resolves a hostname from several candidate sources and uses the first acceptable one. Conceptually:

```text
1. hostname set explicitly in datadog.yaml (or DD_HOSTNAME)
2. hostname_file, if configured
3. container/orchestrator identity, in containerized deployments
4. cloud provider instance metadata (EC2 instance ID, Azure VM ID, and so on)
5. FQDN / operating system hostname
```

Consequences worth internalizing:

```text
Changing the OS hostname creates a NEW Datadog host identity.
Pinning `hostname:` in datadog.yaml makes identity stable but must be unique per host.
Duplicate pinned hostnames merge two machines into one confusing host.
Cloud integrations create their own entities that may not match the Agent host.
```

Compare what the Agent believes with what the OS believes:

```bash
sudo datadog-agent hostname
hostname
hostname -f
```

```powershell
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" hostname
hostname
[System.Net.Dns]::GetHostEntry($env:COMPUTERNAME).HostName
```

If these disagree, the Datadog host name is not a mystery — it is a configuration decision that someone made, possibly by accident.

## Host Aliases

A single host can carry aliases (instance ID, FQDN, short name). Searching only by the name you expect can hide a host that is reporting perfectly well under a different primary name. In the Host List, search by:

```text
short name
FQDN
instance ID
IP-bearing tag
any tag you know should exist (app, site, team)
```

## Host Tag Sources and Precedence

Host tags can arrive from several independent places:

```text
datadog.yaml `tags:` block
DD_TAGS environment variable
cloud provider tags via the cloud integration
container/orchestrator labels
tags applied in the Datadog UI or via API
host tags inherited from integrations
```

To see what the Agent will actually attach:

```bash
sudo datadog-agent tagger-list
```

This is the fastest way to resolve arguments about why a host "has" a tag in one view and not another: host-level tags and metric-level tags are not the same thing, and a tag added in the UI is not visible to the Agent at all.

## Stale, Aged-Out, and Duplicate Hosts

```text
A host that stops reporting eventually falls out of active infrastructure views.
Old host identities linger for a period after a rename before aging out.
Two identities for one machine usually means hostname or metadata changed.
A cloud integration entity is not the same object as an Agent-reporting host.
```

When validating a migration or a rebuild, check the **last reported** timestamp rather than mere presence in a list.

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

## Check the Metric Summary First

Before debugging queries, open the metric in **Metrics → Summary**. It answers several questions at once:

```text
Does the metric exist in this org at all?
Which tags/tag keys are actually attached?
What is the metric type (gauge, count, rate, distribution)?
What unit is declared?
What is the collection interval?
Has it stopped reporting, and when?
How many distinct tag values exist (cardinality)?
```

If the tag key you are filtering on is not listed there, the query was never going to work, regardless of how correct it looks.

## Tag Case and Tag Drift

Tag values collected by the Agent are normalized to lowercase. Values submitted through other paths — API, DogStatsD, custom scripts, imports from a previous monitoring platform — may preserve whatever case they were given.

The result is fragmentation:

```text
site:BOS
site:Bos
site:bos
```

Three tag values, three sets of results, one very confusing dashboard. Normalize to lowercase at the point of submission, regardless of path. See section 28 for the full treatment.

## Metric vs Host vs Monitor Tags

```text
Host tag       attached to the host object; visible in the Host List and on host-tagged metrics
Metric tag     attached to the individual data points at submission time
Monitor tag    metadata on the monitor object, used by search and Notification Rules
```

These three are frequently confused. A tag that exists on a host does not automatically exist on every metric, and neither one puts a tag on a monitor.

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

### Verify what the Agent actually loaded

```bash
sudo datadog-agent configcheck
```

```powershell
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" configcheck
```

Configuration on disk is not the same as configuration in use. `configcheck` shows the instances the Agent loaded and the source of each one.

Expected configuration locations:

```text
Linux    /etc/datadog-agent/datadog.yaml
         /etc/datadog-agent/conf.d/<integration>.d/conf.yaml

Windows  C:\ProgramData\Datadog\datadog.yaml
         C:\ProgramData\Datadog\conf.d\<integration>.d\conf.yaml
```

Common mistakes that produce a silently ignored file:

```text
conf.yml instead of conf.yaml
file placed in conf.d/ rather than conf.d/<integration>.d/
leftover conf.yaml.example still in place and the real file misnamed
file not readable by the Agent user (dd-agent on Linux, ddagentuser on Windows)
tab characters in YAML indentation
unquoted value beginning with a special character
```

### Run the check with more detail

```bash
sudo -u dd-agent datadog-agent check <CHECK_NAME> --log-level debug
sudo -u dd-agent datadog-agent check <CHECK_NAME> --check-rate
```

Flags vary by Agent version. Confirm what your build supports:

```bash
sudo -u dd-agent datadog-agent check --help
```

### Confirm the integration package and version

```bash
sudo datadog-agent integration show <INTEGRATION_NAME>
sudo datadog-agent integration freeze
```

Version matters: metric names, configuration options, and defaults change between integration releases. A configuration copied from current documentation can be invalid for an older installed integration, and a metric that "disappeared" may simply have been renamed upstream.

### Permissions and ownership

```bash
sudo ls -l /etc/datadog-agent/conf.d/<integration>.d/
sudo -u dd-agent cat /etc/datadog-agent/conf.d/<integration>.d/conf.yaml >/dev/null && echo readable
```

If the Agent user cannot read the file, the check never runs — and the Agent may report nothing more specific than the absence of the integration.

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

## Which SNMP Configuration Model Are You Using?

Two models exist, and they fail differently:

```text
A. Explicit instances
   conf.d/snmp.d/conf.yaml with one instance per device or per IP

B. Autodiscovery
   snmp_listener configuration in datadog.yaml scanning subnets with
   one or more credential profiles
```

Diagnostic implications:

```text
Explicit    a device missing means that instance is wrong or unreachable
Discovery   a device missing may mean it was never scanned, never answered,
            or answered with credentials that did not match any configured profile
```

Confirm which model is live:

```bash
sudo datadog-agent configcheck | grep -A20 -i snmp
sudo datadog-agent status | grep -A30 -i "autodiscovery"
```

## Device Namespace

Devices are identified by IP **within a namespace**. The namespace is part of device identity.

```text
Same device, two pollers, same namespace       -> one device object (normal HA behavior)
Same device, two pollers, different namespaces -> two device objects (duplicate inventory)
Namespace changed after onboarding             -> old device object goes stale
```

When running redundant pollers, treat the namespace value as a deliberate design decision and keep it consistent.

## Useful OIDs When Proving Reachability

Walk small, well-known trees before walking an entire device:

```text
1.3.6.1.2.1.1          system group (sysDescr, sysObjectID, sysUpTime, sysName)
1.3.6.1.2.1.1.2        sysObjectID  (drives Datadog profile selection)
1.3.6.1.2.1.2.2        ifTable      (interfaces, 32-bit counters)
1.3.6.1.2.1.31.1.1     ifXTable     (interface names/aliases, 64-bit counters)
```

Example:

```bash
snmpwalk -v2c -c '<community>' <IP> 1.3.6.1.2.1.1
snmpget  -v2c -c '<community>' <IP> 1.3.6.1.2.1.1.2.0
```

If `sysObjectID` does not return, profile troubleshooting is premature — you have a reachability or credential problem.

If `sysObjectID` returns but Datadog applies a generic profile, the device is reachable and the issue is profile mapping instead.

## Credentials Stored in a Secret Backend

If credentials are referenced as `ENC[...]`, an SNMP failure may actually be a secret-resolution failure. Confirm before blaming the network:

```bash
sudo datadog-agent secret
```

See section 27.

## UDP Fails Quietly

SNMP runs over UDP, so a blocked path usually produces a timeout rather than a rejection.

```text
No response can mean: dropped, filtered, wrong credentials, or device ACL denial
```

Validate the path from the **polling Agent's** source address, not from a workstation. A device ACL that permits the old SolarWinds poller will silently ignore a new Datadog poller until the ACL is updated.

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

## Worker Capacity Math

Private Location capacity is a function of test count, frequency, and duration — not of how many tests "feel" small.

```text
Concurrent capacity needed ≈ (tests per minute) × (average test duration in minutes)
```

Symptoms of under-provisioning look like failures rather than like capacity problems:

```text
tests run late or skip intervals
intermittent timeouts under load but not during a manual Fast Test
remaining-slot metric trending toward zero
latency graphs that worsen at scheduled peaks (top of the hour is common)
```

Staggering test frequency across a location is often cheaper than adding workers. Scheduling every test at the same interval concentrates load into the same few seconds of each minute.

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

## Unified Service Tagging

APM, logs, and infrastructure only correlate when the three reserved tags agree:

```text
env
service
version
```

Set them consistently at every layer:

```text
Application   DD_ENV, DD_SERVICE, DD_VERSION (or tracer configuration)
Agent         tags in datadog.yaml / DD_TAGS
Container     labels or pod annotations
```

A service that appears twice in APM under slightly different names is almost always a `DD_SERVICE` mismatch between deployments, not a tracing failure. A trace that exists but has no correlated logs is usually an `env` or `service` mismatch rather than a log pipeline problem.

## Sampling vs Missing

"No traces" and "not the traces I expected" are different problems:

```text
No spans reaching the Agent     -> tracer/network/configuration (this section)
Spans arrive, specific traces absent -> sampling, retention filters, or search window
Service visible, resource absent     -> instrumentation coverage or resource naming
```

Confirm the Agent is receiving spans before changing sampling configuration. The `APM Agent` section of `agent status` reports received and sent trace payloads.

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

## The Four Timing Knobs

Most "the monitor should have fired" arguments come down to these:

```text
Evaluation window     how much data each evaluation considers
Evaluation delay      how long to wait before evaluating, for late-arriving data
New group delay       grace period before a newly seen group can alert
Require full window   whether an evaluation runs when data is incomplete
```

Interactions that surprise people:

```text
Require full window + sparse metric   -> evaluation skipped, monitor stays OK
Evaluation delay too small + cloud metric -> monitor evaluates empty windows
New group delay + short-lived hosts   -> group never becomes eligible to alert
Long window + brief spike             -> average never crosses threshold
```

A monitor that "missed" a spike is often working exactly as configured. Graph the query with the same window and aggregation the monitor uses before assuming a defect.

## Multi-Alert (Grouped) Monitor Lifecycle

A monitor grouped `by {host}` is not one monitor — it is one monitor per group:

```text
Group appears      -> new group delay applies before it can alert
Group alerts       -> notification sent for that group only
Group stops reporting -> the group can go No Data or be dropped after group retention
Group disappears   -> no recovery notification is sent for a group that vanished
```

This explains one of the most common complaints: an alert that never recovers because the group simply stopped existing, and an alert that never fires because the group is brand new.

Check grouping deliberately:

```text
avg:system.cpu.user{env:prod} by {host}      one alert per host
avg:system.cpu.user{env:prod}                one alert overall
```

Grouping also determines what tags the notification carries, which in turn affects routing.

## Read the Monitor's Own History

The monitor status page is evidence, not decoration. It shows:

```text
evaluation results over time
state transitions with timestamps
which groups transitioned
whether evaluations were skipped
downtime/mute overlays
notification events
```

If the history shows no evaluation at the moment you expected an alert, the problem is data or evaluation configuration. If it shows a transition but no delivery, the problem is routing — go to section 20.

## Recovery, Renotify, and Auto-Resolve

```text
Recovery threshold   separate from the alert threshold; prevents flapping
Renotify interval    re-sends while a monitor remains in a triggered state
Auto-resolve         closes a triggered group after a period without data
```

Silence after an alert can mean recovery, renotify being disabled, or auto-resolve quietly closing something that never actually recovered. Verify which one occurred before reporting the incident as resolved.

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

## The Delivery Chain, In Order

Each link produces different evidence:

```text
1. Monitor evaluates and transitions        -> monitor status page / history
2. Notification event is generated          -> Events Explorer
3. Routing decides recipients               -> Notification Rules / message handles
4. Integration accepts the payload          -> integration status, webhook response code
5. Downstream service delivers              -> receiver logs, mail gateway, SMS gateway
6. Human receives it                        -> the only test that actually matters
```

Confirm step 2 independently in the Events Explorer, scoped to the monitor:

```text
source:alert
```

combined with the monitor name or the relevant tags. If the event exists but nobody received anything, the failure is at step 3 or later, and no amount of monitor editing will help.

## Which Tags Routing Actually Matches

This is the most common routing defect:

```text
Monitor tags        metadata on the monitor object      <- Notification Rules match these
Metric query scope  avg:metric{app:abc}                 <- NOT monitor metadata
Host tags           tags on the host object             <- NOT monitor metadata
Group tags          tags of the triggering group        <- behavior differs; verify
```

Putting `app:abc` in the query does not tag the monitor. If your rules key on `app:`, every monitor must carry that tag explicitly on the monitor object.

Verify with the monitor search syntax rather than by eye:

```text
tag:app:abc            monitors carrying the monitor tag app:abc
scope:app:abc          monitors whose query scope includes app:abc
```

The two lists being different is normal — and is exactly the gap that causes silent monitors.

Before a large rollout, confirm empirically whether your rules match group-level tags as well as monitor-level tags. Build one monitor, force one transition, observe the result, and only then scale.

## Conditional Message Blocks

A notification can be suppressed by its own template. The message must include a branch for the state you expect:

```handlebars
{{#is_alert}}      triggered
{{#is_warning}}    warning threshold
{{#is_no_data}}    no data
{{#is_recovery}}   recovered from alert or warning
{{#is_alert_recovery}}   recovered specifically from alert
{{#is_warning_recovery}} recovered specifically from warning
{{#is_alert_to_warning}} severity decreased
```

If every recipient handle sits inside `{{#is_alert}}`, then recoveries and No Data notifications go nowhere, and the monitor looks "broken" only in one direction.

Useful variables in messages:

```text
{{monitor.name}}     {{host.name}}      {{value}}
{{threshold}}        {{comparator}}     {{last_triggered_at}}
{{.name}} / {{.value}} for the triggering group tag
```

## Webhook Debugging

Webhook payloads use `$`-style variables, for example:

```text
$EVENT_TITLE   $EVENT_MSG      $ALERT_TYPE
$ALERT_TRANSITION               $ALERT_STATUS
$HOSTNAME      $TAGS           $PRIORITY
$LINK          $ID             $LAST_UPDATED
$SNAPSHOT      $ORG_ID
```

Debugging order for a custom webhook receiver (a mail/SMS gateway, a ticketing bridge, or a homegrown router):

```text
1. Does the receiver's access log show the request at all?
2. What status code did it return? 2xx or a rejection?
3. Did the payload parse? Log the raw body during testing.
4. Did a required variable arrive empty? Unset variables serialize as empty strings.
5. Did the receiver deliver onward (mail queue, SMS provider, ticket API)?
6. Did downstream filtering discard it (spam rules, quiet hours, dedup)?
```

Test the endpoint independently first:

```bash
curl -i -X POST \
  -H "Content-Type: application/json" \
  -d '{"title":"routing test","msg":"manual test","alert_type":"error"}' \
  https://<webhook-target>
```

If a webhook works with curl but fails from Datadog, look at egress restrictions on the receiver, IP allowlisting, and TLS/certificate requirements.

## Email-Specific Failures

Email is the routing path most likely to fail silently:

```text
recipient address not verified in Datadog
distribution list rejects external senders
mail gateway quarantines the message
SPF/DKIM/DMARC failure on the sending domain
message classified as bulk and delivered to a folder nobody reads
per-recipient rate or size limits
```

"No alert received" is a claim about a human inbox. Confirm the message left Datadog and reached the mail system before concluding that Datadog did not send it.

## Build a Routing Test That Is Safe to Repeat

```text
1. One synthetic monitor whose state you can force deliberately.
2. Exactly the tags used by the routing rule under test.
3. A recipient you control.
4. One transition, observed end to end.
5. One recovery, observed end to end.
6. Then scale, with dry-run output reviewed before any bulk write.
```

Checking routing coverage in bulk afterward is a reporting exercise: list monitors, list rules, and identify monitors that match no rule. A monitor that matches no rule is a silent monitor, and silent monitors are how outages become surprises.

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

### A 403 does not always mean "missing permission"

Datadog can return the same forbidden response for several distinct conditions:

```text
the application key lacks the required scope
the key owner's role lacks the underlying permission
the resource itself is restricted
the product is not enabled for the organization or site
the endpoint does not exist for that site
```

Read the response body, not just the status code. Wording that references failed permission authorization checks points to a permissions problem. An empty or generic body on an endpoint that works elsewhere often points to product availability or site differences instead — which no amount of scope-granting will fix.

Capture the full response while diagnosing:

```bash
curl -i -X GET \
  -H "DD-API-KEY: $DD_API_KEY" \
  -H "DD-APPLICATION-KEY: $DD_APP_KEY" \
  "https://<DATADOG_API_SITE>/api/v2/..." | tee /tmp/dd-403.txt
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

# 22.5 Other Status Codes Worth Recognizing

```text
404  wrong path, wrong API version, wrong site, or object deleted/never existed
405  correct path, wrong HTTP method
409  conflict — object already exists, or concurrent modification
413  payload too large — batch size too aggressive
422  request understood but semantically invalid for this resource
500  Datadog-side error; retry with backoff, then escalate with request details
502/503/504  transient gateway/availability issue or a proxy in the middle
```

Useful habits for automation:

```text
Treat 4xx as "fix the request"; treat 5xx and 429 as "retry with backoff".
Log the request ID/response headers on failure, not just the body.
Never retry a non-idempotent POST blindly — check whether the object was created.
Make create operations create-or-update (lookup, diff, then PATCH only on change).
Always support a dry-run mode before any bulk write.
```

A 409 during a bulk migration usually means the script is not idempotent, not that Datadog is misbehaving.

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

## Local and Remote Flares

Generate a flare without sending it, for review under a data-handling policy:

```bash
sudo datadog-agent flare --local
```

The archive is written locally; upload it manually after review.

If Remote Configuration and Fleet Automation are available and supported for your site and Agent version, a flare can also be requested remotely from the Datadog UI, which is useful when shell access requires a change request.

## Raise Log Level Before Collecting

A flare taken while the Agent is logging at `info` may not contain the detail support needs. When the problem is reproducible:

```bash
sudo datadog-agent config set log_level debug
# reproduce the problem
sudo datadog-agent flare <CASE_ID>
sudo datadog-agent config set log_level info
```

Note the exact reproduction time so support can find it in the archive.

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

# 25. Custom Metrics and DogStatsD Not Arriving

Custom metrics submitted from application code, scripts, or automation follow a different path than integration metrics, and they fail in different ways.

## Symptoms

```text
Script reports success but the metric never appears
Metric appears briefly and then stops
Counts are lower than expected
Metric appears without expected tags
Metric arrives from one host but not from a container
Metric name exists but has no recent points
```

## Checks

Identify the submission path first:

```text
A. DogStatsD to the Agent (UDP 8125 or a Unix socket)
B. HTTP API submission directly to Datadog
C. Agent custom check (Python check in checks.d)
D. Tracer/library metrics (runtime metrics, custom spans metrics)
```

Each has a different failure mode:

```text
A  packets silently dropped; nothing errors
B  HTTP status code tells you what happened
C  appears in `agent status` under Running Checks
D  depends on tracer configuration and Agent APM configuration
```

## Commands

DogStatsD statistics from the Agent:

```bash
sudo datadog-agent status | grep -A20 -i dogstatsd
sudo datadog-agent dogstatsd-stats
```

Send a test metric by hand:

```bash
echo -n "test.metric.manual:1|c|#env:prod,source:manual" | nc -u -w1 127.0.0.1 8125
```

PowerShell equivalent:

```powershell
$u = New-Object System.Net.Sockets.UdpClient
$b = [Text.Encoding]::ASCII.GetBytes("test.metric.manual:1|c|#env:prod,source:manual")
$u.Send($b, $b.Length, "127.0.0.1", 8125)
$u.Close()
```

Then search Metrics Explorer for:

```text
test.metric.manual
```

Direct API submission test:

```bash
curl -i -X POST "https://api.<DATADOG_SITE>/api/v2/series" \
  -H "Content-Type: application/json" \
  -H "DD-API-KEY: $DD_API_KEY" \
  -d '{"series":[{"metric":"test.metric.api","type":3,"points":[{"timestamp":'"$(date +%s)"',"value":1}],"tags":["env:prod","source:manual"]}]}'
```

A `202` means accepted for processing — not that the point is queryable yet.

## Likely Causes

```text
UDP packets dropped: buffer too small, high volume, or no listener
DogStatsD disabled in the Agent
Container sends to localhost instead of the Agent host/socket
dogstatsd_non_local_traffic not enabled when traffic is not local
Port 8125 blocked or already in use
Metric name contains invalid characters
Counter vs gauge semantics misunderstood
Tag cardinality causes rate limiting or truncation
Short-lived process exits before flush
Custom check raises an exception and never submits
API submission returns 2xx but with the wrong timestamp units
```

## Resolution

```text
Packet loss        increase receive buffer / use Unix socket / batch submissions
Container path     point the client at the Agent host or a mounted socket
Non-local traffic  enable it explicitly and restrict access at the network layer
Short-lived jobs   flush before exit, or submit via API instead of DogStatsD
Cardinality        remove unbounded tag values (request IDs, timestamps, full URLs)
Naming             use lowercase, dots for namespacing, no spaces or high-cardinality suffixes
```

Unbounded tags are the most expensive mistake in this section. A tag value derived from a user ID, a request ID, or a timestamp turns one metric into millions of time series, and the consequences are billing as well as performance.

## Validation

```text
dogstatsd-stats shows packets received and no growing error counters
manual test metric appears in Metrics Explorer within a normal delay
expected tags are present in Metric Summary
counts align with a known-quantity test (submit exactly 10, expect 10)
```

---

# 26. Cloud Integration Metrics Missing (AWS / Azure)

Cloud metrics are collected by Datadog's crawlers through the cloud provider's own monitoring API. There is no Agent in this path, so Agent troubleshooting does not apply.

## Symptoms

```text
Cloud resource exists but no metrics in Datadog
Some services report, others do not
Metrics arrive late
Metrics stop after an IAM/role change
Resource tags missing from Datadog metrics
Cloud account appears configured but no data
```

## Checks

```text
Is the integration tile configured for the correct account/subscription?
Does the cross-account role or app registration still exist?
Are the required read permissions still attached?
Is the specific service/namespace enabled in the integration configuration?
Do account-level tag filters exclude this resource?
Does the resource actually publish the metric in the provider's own console?
Is the metric delayed rather than missing?
Are provider-side API limits being hit?
```

Verify at the source first:

```bash
aws cloudwatch list-metrics --namespace AWS/EC2 --region us-east-1 \
  --dimensions Name=InstanceId,Value=<INSTANCE_ID>

aws cloudwatch get-metric-statistics --namespace AWS/EC2 \
  --metric-name CPUUtilization --region us-east-1 \
  --dimensions Name=InstanceId,Value=<INSTANCE_ID> \
  --start-time <ISO8601> --end-time <ISO8601> --period 300 --statistics Average
```

If the provider does not have the data, Datadog cannot have it either.

## Likely Causes

```text
role/app registration deleted, expired, or re-scoped
required permissions removed by a security policy change
metric namespace not enabled in the integration configuration
tag-based filtering excludes the resource
resource is in a region or subscription that was never added
detailed/enhanced monitoring not enabled on the resource
provider API throttling the crawler
metric genuinely delayed (crawler-based collection is not real time)
expecting an Agent-only metric from a cloud-only resource
```

The last one is common: memory and disk metrics for an EC2 instance generally require an Agent. The cloud provider does not publish them by default.

## Resolution

```text
Permissions      restore the documented read permissions; re-validate the integration tile
Namespaces       enable only the services you actually need
Tag filters      confirm include/exclude expressions and their case sensitivity
Delay            set a monitor evaluation delay appropriate for crawler latency
Coverage         install the Agent where host-level detail is required
Throttling       reduce namespace scope or spread accounts across integrations
```

For monitors on crawler-collected metrics, an evaluation delay of at least 15 minutes is a common baseline. Without it, monitors evaluate windows that were always going to be empty.

## Validation

```text
integration tile reports no errors
expected namespaces enabled
metrics visible in Metrics Explorer with provider tags attached
timestamps consistent with provider-side delay, not older
monitors on these metrics stop producing spurious No Data
```

---

# 27. Secret Backend (`ENC[]`) Resolution Failures

When credentials are referenced as `ENC[...]`, a credential failure may be a secret-resolution failure rather than a wrong password.

## Symptoms

```text
Integration reports authentication failure with credentials known to be correct
Agent log shows secret backend errors at startup
Check configuration loads but the credential field is empty
Behavior differs between a manual check run and the running Agent
Works on one host, fails on another with identical configuration
```

## Checks

```text
Is secret_backend_command configured and pointing at an existing executable?
Are the file permissions and ownership correct for the Agent user?
Does the backend return valid JSON for every requested handle?
Does the executing identity have access to the secret store?
Is the token/credential used by the backend itself still valid?
Is the handle name spelled exactly as stored?
```

## Commands

```bash
sudo datadog-agent secret
```

```powershell
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" secret
```

This reports the configured backend, permission verification results, and which handles resolved. Treat it as the authoritative answer to "did the Agent actually get the secret?"

Permission expectations:

```text
Linux    executable owned by root (or the documented owner), not writable by others,
         executable by the Agent user, typically mode 500/700 as documented

Windows  executable readable/executable only by ddagentuser and the Administrators group;
         inherited permissions are a common cause of refusal to run the backend
```

The Agent deliberately refuses to run a secret backend with loose permissions. A refusal is a safety feature, not a bug.

## Likely Causes

```text
executable path wrong or not present on this host
permissions too permissive, so the Agent refuses to execute it
Agent user cannot execute the backend
backend returns malformed JSON or a non-zero exit code
backend depends on an environment variable the Agent service does not have
underlying store unreachable (network/DNS/token expiry)
secret rotated in the store but the handle name changed
container image missing the backend binary or its dependencies
```

Environment differences deserve special attention: a backend that works when you run it interactively may fail under the service account, which has a different environment, different proxy settings, and no interactive session.

## Resolution

```text
1. Run `datadog-agent secret` and read the permission verification output.
2. Correct ownership/permissions to the documented values.
3. Execute the backend manually AS THE AGENT USER and inspect stdout.
4. Validate the JSON shape returned for a single handle.
5. Confirm the store is reachable from the host and the auth token is valid.
6. Restart the Agent and re-check.
```

Manual execution pattern on Linux:

```bash
echo '{"version":"1.0","secrets":["<handle>"]}' | sudo -u dd-agent /path/to/secret-backend
```

## Validation

```text
`datadog-agent secret` lists the handle as resolved
the dependent check authenticates successfully
no secret-backend errors in agent.log after restart
rotation test: rotate in the store, restart, confirm the new value is used
```

---

# 28. Tag Problems and Tag Fragmentation

Tags are the join key for everything in Datadog: queries, dashboards, monitors, routing, cost attribution, and access control. Tag problems rarely announce themselves — they show up as missing data, partial dashboards, and alerts that go to nobody.

## Symptoms

```text
Query returns fewer hosts than expected
Dashboard populates for one team and not another
Notification Rule matches nothing
Two tag values that should be one
Tag visible on the host but not on the metric
Group-by produces an unexpected number of groups
Cost/usage attribution incomplete
```

## The Case Problem

Agent-side ingestion normalizes tag values to lowercase. Submissions through other paths may preserve case.

```text
site:BOS   from an import script
site:Bos   from a spreadsheet
site:bos   from the Agent
```

These do not merge. Queries filtering on one miss the others, and no error is produced anywhere.

```text
Normalize to lowercase at submission time, on every path:
API submissions, DogStatsD, imports, migration scripts, UI-applied host tags.
```

Validate after any bulk import:

```text
Metrics Explorer -> group by the tag key and look for near-duplicate values
Metric Summary   -> inspect the full tag value list for a representative metric
```

## Where a Tag Lives Matters

```text
Host tag      on the host object            affects host-scoped queries and host tag views
Metric tag    on the data points            affects metric queries
Monitor tag   on the monitor object         affects monitor search and Notification Rules
Log tag       on log events / pipelines     affects log queries and log-based monitors
Span tag      on traces                     affects APM search and trace metrics
Integration tag attached by the integration configuration
```

A tag applied in the Datadog UI to a host does not retroactively appear on metric points already submitted, and it does not appear on monitors at all.

## Reserved and Structural Tags

```text
host      identity join key across products
env       unified service tagging
service   unified service tagging
version   unified service tagging
```

Overloading or misusing these breaks correlation between infrastructure, APM, and logs in ways that are tedious to unwind later.

## Cardinality

```text
Good tag values:   bounded, meaningful, stable  (env, app, site, team, role)
Bad tag values:    unbounded or unique per event (request_id, timestamp, full URL, PID)
```

High cardinality drives custom-metric counts, slows queries, and can silently truncate what you see.

## Checks

```text
Does the tag key exist on this data type at all?
Is the value spelled and cased exactly as queried?
Is it applied at the right layer (host vs metric vs monitor)?
Did it exist during the time range being queried?
Is it applied consistently across every collection path?
Is any object missing a tag your routing depends on?
```

## Commands

Agent-side view:

```bash
sudo datadog-agent tagger-list
sudo datadog-agent status | grep -A20 -i "host tags"
```

Inventory-style audit via the API (adapt to the object type):

```bash
curl -s -H "DD-API-KEY: $DD_API_KEY" -H "DD-APPLICATION-KEY: $DD_APP_KEY" \
  "https://api.<DATADOG_SITE>/api/v1/monitor" \
  | python -c "import sys,json;[print(m['id'], m['name'], m.get('tags')) for m in json.load(sys.stdin)]"
```

## Resolution

```text
1. Decide the authoritative tag schema and write it down: keys, allowed values, casing.
2. Enforce it at submission, not by cleanup afterwards.
3. Remediate existing objects deliberately, with a dry run first.
4. Re-audit after every bulk change and after every onboarding wave.
5. Gate automation on compliance rather than discovering drift months later.
```

Tag governance is much cheaper to establish before an estate is onboarded than to retrofit afterwards, because every dashboard, monitor, and routing rule written in the meantime encodes the inconsistency.

## Validation

```text
grouping by the tag key yields the expected value set, with no near-duplicates
routing rules match the expected monitor population
dashboards populate for every team, not just the first one onboarded
no object that should be routed is missing its routing tag
```

---

# 29. Data Is Present but Wrong

Not every incident is missing data. Some are data that arrives and is then misread — by a query, a widget, or a human.

## Symptoms

```text
Graph shows a number nobody believes
Value differs between two widgets using "the same" metric
Sum across groups does not match the total
Counts look too low at wide time ranges
Percentages exceed 100
Spike disappears when the time range widens
Two dashboards disagree about the same hour
```

## Metric Types

```text
GAUGE          last value in the interval; averaging across time is meaningful
COUNT          number of events in the interval; summing is meaningful
RATE           per-second value; comparing to a count directly is not meaningful
DISTRIBUTION   percentile-capable; aggregation rules differ from gauges
```

Using the wrong aggregation for the type is the most common source of wrong-looking numbers.

```text
.as_count()    interpret as raw counts
.as_rate()     interpret as per-second
sum vs avg     across groups, these answer different questions
```

## Rollups Hide Spikes

At wide time ranges, points are aggregated into buckets. The default aggregation can smooth away exactly the spike you are investigating.

```text
.rollup(max, 60)   preserve peaks
.rollup(avg, 60)   smooth
.rollup(sum, 60)   totals
```

If a spike is visible at a 1-hour window and invisible at 1 week, the data did not change — the bucket aggregation did.

## Aggregation Across Space vs Time

```text
avg:metric{*}            averages across all matching series (space)
.rollup(avg, 300)        averages within each bucket (time)
avg:metric{*} by {host}  no cross-host averaging; one series per host
```

A dashboard that averages across hosts will understate a single hot host, and a monitor written the same way will never fire for it.

## Interpolation and Sparse Data

```text
default_zero()   treat missing as zero — only when zero is semantically correct
Sparse series    can appear to "drop to nothing" purely because of a gap
```

Applying `default_zero()` to a metric where absence means "not collected" (rather than "nothing happened") manufactures data that never existed.

## Units and Scale

```text
bytes vs bits              factor of 8, usually noticed only on a network graph
bytes vs kilobytes         off by 1024, usually noticed by a capacity planner
percent as 0-1 vs 0-100    a graph that tops out at 1 or at 10,000
counters vs derived rates  SNMP counters especially
```

Check the declared unit in Metric Summary before rescaling anything in a formula. Correcting units in a widget formula while the underlying metric is already correct doubles the error.

## Counter Wraps and Resets

```text
SNMP 32-bit counters wrap on busy interfaces; prefer 64-bit (ifXTable) counters.
A process restart resets a monotonic counter to zero.
A sudden impossible negative or enormous delta usually means a reset or a wrap,
not a real event.
```

## Resolution

```text
1. Open the metric in Metric Summary: type, unit, interval.
2. Reproduce the number in Metrics Explorer with the simplest possible query.
3. Add one transformation at a time and watch when the number changes.
4. Compare against the source system independently.
5. Fix the query or the submission, then fix every copy of it.
```

## Validation

```text
the same query returns the same value in Explorer, dashboard, and monitor
totals reconcile with the source system
the value behaves sensibly at 1 hour, 1 day, and 1 week
peaks survive at wide time ranges when that matters
```

---

# 30. Downtimes, Mutes, and Suppressed Alerts

An alert that never arrives is not always a delivery failure. Sometimes something deliberately suppressed it, possibly months ago.

## Symptoms

```text
Monitor shows Alert in the UI but nothing was sent
Alerts stopped for one team or one site only
Alerts resumed unexpectedly
Maintenance window ended but alerts stayed silent
Monitor muted with no obvious owner
A new host inherits silence it should not have
```

## Checks

```text
Is there an active downtime matching this monitor or its scope?
Is the monitor itself muted?
Is the host muted?
Was a downtime created with a scope broader than intended?
Is the downtime recurring, and did the recurrence outlive its purpose?
Did a bulk operation mute a large set of monitors?
Does a downtime scope use a tag that matches more than expected?
```

## Scope Is the Usual Culprit

```text
Downtime scope: env:prod             every production monitor
Downtime scope: host:app-01          one host across all monitors
Downtime scope: app:abc AND site:bos narrow and intentional
Downtime scope: *                    everything, including things you forgot
```

A downtime scoped to a tag will silence anything that later acquires that tag. Onboarding a new host into `app:abc` can hand it a pre-existing silence nobody remembers creating.

## Commands

List downtimes through the API and inspect scope and expiry:

```bash
curl -s -H "DD-API-KEY: $DD_API_KEY" -H "DD-APPLICATION-KEY: $DD_APP_KEY" \
  "https://api.<DATADOG_SITE>/api/v1/downtime" \
  | python -m json.tool
```

Look specifically for:

```text
active/disabled state
scope
start and end (an empty end is indefinite)
recurrence rules
creator and creation date
```

## Likely Causes

```text
indefinite downtime created during an incident and never removed
recurring maintenance window with an overly broad scope
monitor muted individually during triage
host muted during patching and never unmuted
downtime created by automation whose cleanup step failed
notification suppressed by a monitor option rather than a downtime
```

## Resolution

```text
1. Identify every downtime whose scope matches the silent monitor.
2. Determine intent and owner before cancelling anything.
3. Narrow scopes rather than deleting suppression wholesale.
4. Give every downtime an explicit end time unless there is a documented reason.
5. Review indefinite downtimes on a schedule.
```

Indefinite, broadly-scoped downtimes are one of the quietest ways for monitoring coverage to decay over time.

## Validation

```text
force a safe transition and confirm delivery
confirm the downtime list contains no unexplained indefinite entries
confirm scheduled maintenance windows match their intended scope
confirm newly onboarded hosts are not inheriting stale suppression
```

---

# 31. Network Path and Endpoint Reference

Use this when writing firewall rules, proxy exceptions, or TLS inspection bypasses — and when proving that the network is or is not the problem.

**Always verify against the current documentation for your exact site before making a change.** Endpoint names are site-specific, and the set grows as products are added.

## Site Substitution

Replace `<SITE>` with your Datadog site domain:

```text
US1    datadoghq.com
US3    us3.datadoghq.com
US5    us5.datadoghq.com
EU1    datadoghq.eu
AP1    ap1.datadoghq.com
US1-FED ddog-gov.com
```

## Destinations by Product

```text
Metrics, service checks, events, Agent metadata   <VERSION>-app.agent.<SITE>
                                                  (allowlist *.agent.<SITE>)
Agent API calls (key validation, etc.)            api.<SITE>
Agent flare                                       <VERSION>-flare.agent.<SITE>
Remote Configuration / Fleet Automation           config.<SITE>
APM                                               trace.agent.<SITE>
                                                  instrumentation-telemetry-intake.<SITE>
Profiling                                         intake.profile.<SITE>
Live processes / containers                       process.<SITE>
Orchestrator (Kubernetes)                         orchestrator.<SITE>
                                                  contlcycle-intake.<SITE>
Container images                                  contimage-intake.<SITE>
Network Device Monitoring                         ndm-intake.<SITE>
SNMP traps                                        snmp-traps-intake.<SITE>
NetFlow                                           ndmflow-intake.<SITE>
Network Path                                      netpath-intake.<SITE>
Database Monitoring                               dbm-metrics-intake.<SITE>
                                                  dbquery-intake.<SITE>
Logs (HTTP)                                       agent-http-intake.logs.<SITE>
Logs (TCP)                                        agent-intake.logs.<SITE>
Synthetics Private Location worker                intake.synthetics.<SITE>
```

Agent installation and package repositories:

```text
install.datadoghq.com
apt.datadoghq.com
yum.datadoghq.com
keys.datadoghq.com
windows-agent.datadoghq.com
```

The metrics intake hostname includes the Agent version, for example `7-50-0-app.agent.datadoghq.com`. This is why an allowlist must use `*.agent.<SITE>` rather than a single literal hostname: **an Agent upgrade changes the hostname it connects to**, and a literal-hostname firewall rule breaks silently on upgrade.

## Ports

Outbound:

```text
443/tcp    most Agent traffic (metrics, APM, processes, containers, NDM, remote config)
10516/tcp  log collection over TCP (when not using HTTP)
123/udp    NTP
```

Inbound, local to the host only:

```text
8125/udp   DogStatsD (localhost unless dogstatsd_non_local_traffic is enabled)
8126/tcp   APM trace receiver
5000/tcp   go_expvar
5001/tcp   Agent IPC API
5002/tcp   Agent browser GUI
6062/tcp   Process Agent debug endpoints
6162/tcp   Process Agent runtime settings
```

These local ports should not be exposed to untrusted networks. An open trace receiver or DogStatsD port accepts data from anyone who can reach it.

## IP-Based Allowlisting

If policy requires IP ranges rather than hostnames, Datadog publishes per-product IP range files:

```text
https://ip-ranges.<SITE>/
https://ip-ranges.<SITE>/logs.json
https://ip-ranges.<SITE>/apm.json
```

Allowlist the full published set rather than the subset currently in use; the active addresses vary over time within that set. Automate the refresh — a static copy of an IP list becomes an outage with a delayed fuse.

## Proving the Path

```bash
# DNS
dig +short api.<SITE>

# TCP reachability
nc -vz api.<SITE> 443

# TLS chain as actually presented
openssl s_client -connect api.<SITE>:443 -servername api.<SITE> </dev/null 2>/dev/null \
  | openssl x509 -noout -issuer -subject -dates

# Through the configured proxy
curl -v -x http://proxy.example.com:3128 https://api.<SITE>/api/v1/validate \
  -H "DD-API-KEY: $DD_API_KEY"
```

```powershell
Resolve-DnsName api.<SITE>
Test-NetConnection api.<SITE> -Port 443
Invoke-WebRequest "https://api.<SITE>/api/v1/validate" -Headers @{ "DD-API-KEY" = $env:DD_API_KEY } -UseBasicParsing
```

A successful `/api/v1/validate` proves DNS, routing, TLS, proxy, and key validity in a single call. It is the fastest single test in this guide.

---

# 32. Windows-Specific Notes

Windows hosts fail in ways Linux hosts do not, and the differences cost time at 2 AM.

## Paths

```text
Binary          C:\Program Files\Datadog\Datadog Agent\bin\agent.exe
Main config     C:\ProgramData\Datadog\datadog.yaml
Check configs   C:\ProgramData\Datadog\conf.d\<integration>.d\conf.yaml
Logs            C:\ProgramData\Datadog\logs\agent.log
                C:\ProgramData\Datadog\logs\trace-agent.log
                C:\ProgramData\Datadog\logs\process-agent.log
```

`C:\ProgramData` is hidden by default in Explorer, which is why people report that "the configuration directory does not exist."

## Services

```text
datadogagent            core Agent
datadog-trace-agent     APM
datadog-process-agent   live processes and containers
datadog-system-probe    network/system probe, when enabled
```

```powershell
Get-Service *datadog*
Get-Service datadogagent | Select-Object Name, Status, StartType
Restart-Service datadogagent
```

## The Agent User

The Agent runs as `ddagentuser` by default. Consequences:

```text
It must have read access to every configuration file it loads.
It must have access to any log file you ask it to tail.
It must have the right to query the resources an integration targets
(WMI, performance counters, remote systems, and so on).
A domain policy that resets local account rights can break collection silently.
```

Grant access explicitly rather than by adding the account to broad groups.

## Event Log and Performance Counters

```text
Windows Event Log collection is configured through the Agent's event log integration.
Performance counter checks depend on healthy counters on the host itself.
```

Rebuild corrupted counters at the OS level before troubleshooting Datadog:

```powershell
lodctr /R
```

If the counters are broken, every counter-based integration reports nonsense or nothing, and the Agent is merely the messenger.

## PowerShell Quoting Traps

```powershell
# Use the call operator for quoted paths
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" status

# Backtick is the line continuation character, not backslash
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" `
  check snmp

# Environment variables set in a session are not visible to the service
```

That last point causes real confusion: setting `$env:DD_API_KEY` in a console does not change what the Agent service uses. Service-level configuration lives in `datadog.yaml` or in machine-level environment variables, and requires a service restart.

## Time

```powershell
w32tm /query /status
w32tm /query /source
w32tm /resync
```

Domain-joined machines should follow the domain hierarchy. A VM that syncs from both a hypervisor and a domain controller can drift in ways that look like intermittent Datadog failures.

---

# 33. Migration-Era Pitfalls

Guidance for teams moving from a legacy platform (SolarWinds Orion or similar) to Datadog, where parity — not novelty — is the objective.

## Identity and Naming

```text
Legacy node names may not match the Agent hostname Datadog derives.
Interface indexes are not stable identifiers across reboots or firmware changes.
Device IP is identity in NDM, within a namespace; plan the namespace deliberately.
A rename during migration creates a second identity rather than updating the first.
```

Decide and document the identity strategy **before** bulk onboarding, because monitors, dashboards, and routing all encode it.

## Imported Tag Values

Data exported from a legacy system rarely matches the casing and vocabulary you want in Datadog.

```text
Normalize to lowercase before submission.
Map legacy custom properties to a deliberate tag schema rather than copying keys verbatim.
Reject values that do not match the schema instead of importing them "for now".
```

See section 28. Every inconsistency imported during migration will be encoded into monitors and routing within days.

## Alert Parity Is Not Alert Translation

```text
Legacy threshold semantics rarely map one-to-one onto Datadog evaluation windows.
A legacy "node down" check is usually a service check or host monitor in Datadog,
not a metric threshold.
Legacy dependency suppression has no automatic equivalent; plan explicitly.
Legacy mute/unmanage windows map to downtimes with scopes, not to per-object flags.
```

Translate intent, not configuration. A faithful copy of a legacy alert often produces either constant noise or silence.

## Parallel Run

```text
Run both platforms for a defined overlap period.
Compare alert volume per application and per site, not just in aggregate.
Investigate every alert that fires in one platform and not the other.
Track coverage explicitly: which objects are monitored in old but not new?
Record the cutover criteria before the cutover week, not during it.
```

The two most valuable artifacts from a parallel run are a list of objects monitored in the old system with no equivalent in the new one, and a list of monitors in the new system that match no routing rule.

## Routing Coverage

```text
Every monitor should match at least one notification rule.
Every application team should have a verified recipient.
Every rule should be tested with one real transition before bulk enablement.
Recovery paths deserve the same testing as alert paths.
```

A monitor that fires correctly and notifies nobody is worse than no monitor, because it creates the appearance of coverage.

## Decommission Checklist

```text
[ ] All objects onboarded and reporting in Datadog
[ ] Tag compliance verified across hosts, monitors, synthetics, dashboards
[ ] Every monitor matched to a notification rule
[ ] Alert and recovery paths tested per application team
[ ] Dashboards rebuilt and validated by their owners
[ ] Parallel-run discrepancies resolved or explicitly accepted
[ ] Credentials rotated out of the legacy platform
[ ] Legacy polling disabled before decommission, not after
[ ] Historical data retention/export requirements satisfied
[ ] Rollback plan documented for the cutover window
```

Disable legacy polling before decommissioning the legacy platform. Devices with ACLs permitting only the legacy poller are discovered at that moment, not later.

---

# 34. Fast Command Reference

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

## Agent - Diagnostics and Configuration

```bash
sudo datadog-agent diagnose
sudo datadog-agent configcheck
sudo datadog-agent config
sudo datadog-agent config set log_level debug
sudo datadog-agent config set log_level info
sudo datadog-agent tagger-list
sudo datadog-agent workload-list
sudo datadog-agent secret
sudo datadog-agent dogstatsd-stats
sudo datadog-agent stream-logs
sudo datadog-agent integration show <INTEGRATION>
sudo datadog-agent integration freeze
sudo datadog-agent status -j
sudo datadog-agent flare --local
```

Windows equivalents use the same subcommands:

```powershell
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" configcheck
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" tagger-list
& "$env:ProgramFiles\Datadog\Datadog Agent\bin\agent.exe" secret
```

Available subcommands vary by Agent version:

```bash
sudo datadog-agent --help
```

## Docker

```bash
docker ps
docker ps -a
docker logs <container>
docker inspect <container>
docker stats <container>
docker exec -it <datadog-agent> agent status
docker exec -it <datadog-agent> agent configcheck
docker exec -it <datadog-agent> agent tagger-list
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

## API Smoke Tests

```bash
# Validate key, site, DNS, TLS, and proxy in one call
curl -s -o /dev/null -w "%{http_code}\n" \
  -H "DD-API-KEY: $DD_API_KEY" \
  "https://api.<DATADOG_SITE>/api/v1/validate"

# Inspect rate-limit headers
curl -i -H "DD-API-KEY: $DD_API_KEY" -H "DD-APPLICATION-KEY: $DD_APP_KEY" \
  "https://api.<DATADOG_SITE>/api/v1/monitor" | head -40
```

```powershell
$h = @{ "DD-API-KEY" = $env:DD_API_KEY }
Invoke-WebRequest "https://api.<DATADOG_SITE>/api/v1/validate" -Headers $h -UseBasicParsing
```

## Network

Linux:

```bash
curl -vk https://<target>
nslookup <target>
dig <target>
ss -lntp
ip route
openssl s_client -connect <target>:443 -servername <target> </dev/null
nc -vz <target> 443
```

Windows:

```powershell
Test-NetConnection <target> -Port 443
Resolve-DnsName <target>
Get-NetTCPConnection
```

---

# 35. Troubleshooting Decision Trees

## 35.1 "No Data"

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

## 35.2 "Host Down"

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

## 35.3 "SNMP Device Missing"

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

## 35.4 "Monitor Alert, No Notification"

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

## 35.5 "APM Missing"

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

## 35.6 "Custom Metric Missing"

```text
Custom metric absent
       │
       ▼
Submitted via DogStatsD or API?
   ┌───┴───┐
DogStatsD  API
   │        │
   ▼        ▼
dogstatsd-  HTTP status
stats shows  code?
packets?     │
   │      ┌──┴──┐
 ┌─┴─┐   2xx   4xx
NO   YES  │      │
 │    │   ▼      ▼
 ▼    ▼  wait/  fix request
client Agent   query    (auth/shape/
/port  received          timestamp)
issue  it → tag,
       name, or
       query issue
```

## 35.7 "Tag Mismatch"

```text
Query/rule matches nothing
         │
         ▼
Does the tag key exist on this data type?
    ┌────┴────┐
   NO        YES
    │         │
    ▼         ▼
Applied at  Values match exactly
wrong layer  (including case)?
(host vs      ┌────┴────┐
metric vs    NO        YES
monitor)      │         │
              ▼         ▼
        normalize   time range /
        and re-     permission /
        submit      object scope
```

---

# 36. Escalation Evidence Template

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
Proxy in path: Yes / No
TLS inspection in path: Yes / No / Unknown
Clock offset from agent status:

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

Relevant tags on the affected object:
Notification rule expected to match:
Downtime/mute checked: Yes / No

Flare collected:
Support case ID:
```

---

# 37. Official References

Use these as the authoritative starting points because Datadog behavior evolves.

## Agent

- Agent Troubleshooting  
  https://docs.datadoghq.com/agent/troubleshooting/

- Network Traffic (endpoints and ports)  
  https://docs.datadoghq.com/agent/configuration/network/

- Agent Proxy Configuration  
  https://docs.datadoghq.com/agent/configuration/proxy/

- Secrets Management  
  https://docs.datadoghq.com/agent/configuration/secrets-management/

- Datadog Sites  
  https://docs.datadoghq.com/getting_started/site/

- Datadog Status Page  
  https://status.datadoghq.com

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

## Metrics, Tags, and Queries

- Metric Types  
  https://docs.datadoghq.com/metrics/types/

- Getting Started with Tags  
  https://docs.datadoghq.com/getting_started/tagging/

- Using Tags  
  https://docs.datadoghq.com/getting_started/tagging/using_tags/

- Unified Service Tagging  
  https://docs.datadoghq.com/getting_started/tagging/unified_service_tagging/

- Rollup and Query Functions  
  https://docs.datadoghq.com/dashboards/functions/rollup/

- DogStatsD  
  https://docs.datadoghq.com/developers/dogstatsd/

## Cloud Integrations

- Amazon Web Services  
  https://docs.datadoghq.com/integrations/amazon_web_services/

- Microsoft Azure  
  https://docs.datadoghq.com/integrations/azure/

## Monitor Suppression

- Downtimes  
  https://docs.datadoghq.com/monitors/downtimes/

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
