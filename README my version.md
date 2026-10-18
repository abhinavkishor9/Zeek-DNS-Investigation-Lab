# Zeek-DNS-Investigation-Lab
## Overview
Detection engineering is the process of designing, testing, validating, and improving security detection logic so that suspicious activities can be identified from available telemetry.
In this lab, the focus is on developing and validating a Wazuh detection for Windows process-creation activity collected through Sysmon.
The goal is not simply to generate an event and confirm that it appears in a dashboard. We need to understand what activity the detection is intended to identify, which event fields support the detection, how the rule behaves during testing, and whether its results are accurate.

This project documents a hands-on DNS investigation using Zeek 8.0.10 and an offline packet capture in Ubuntu 24.04 running under WSL.

The investigation began with a packet capture that did not contain DNS transactions. A separate capture was then created using `tcpdump` while DNS queries were generated with `dig`. During analysis, Zeek reported a possible UDP checksum issue and did not initially generate `dns.log`. Reprocessing the capture with checksum validation disabled using the `-C` option successfully generated the DNS log.

The lab demonstrates practical DNS log analysis, packet capture validation, checksum troubleshooting, and evidence-based investigation documentation.

## Objectives
- Analyze an offline packet capture with Zeek to identify and investigate DNS transactions.
- Validate packet capture contents and distinguish DNS traffic from unrelated protocols before beginning the investigation.
- Generate controlled DNS queries and capture network traffic using `dig` and `tcpdump`.
- Extract and interpret queried domain names, DNS query types, response codes, returned answers, timestamps, and connection identifiers from `dns.log`.
- Use `zeek-cut`, `sort`, and `uniq` to summarize DNS query frequencies, query types, and response-code distributions.
- Investigate Zeek's UDP checksum warning and compare standard analysis with checksum-tolerant processing using the `-C` option.
- Correlate DNS records with connection metadata to investigate endpoint attribution and the relationship between DNS transactions and network connections.
- Identify evidence limitations, distinguish observed behavior from suspicious patterns, and avoid unsupported conclusions about malicious activity.
- Document investigation findings, troubleshooting decisions, and outstanding questions in a structured SOC investigation report.## Lab Environment

| Component | Details |
|---|---|
| Operating system | Ubuntu 24.04 on WSL |
| Network analysis tool | Zeek 8.0.10 |
| Packet capture tool | tcpdump |
| DNS query utility | dig |
| Analysis mode | Offline PCAP |
| Working directory | `~/zeek-labs/lab03` |
| Primary output | `dns.log` |
| Supporting output | `conn.log`, `packet_filter.log` |

## Investigation Scenario
### Scenario

You are working as a SOC Analyst investigating DNS activity in a controlled network environment. Your objective is to examine DNS transactions captured in a packet capture (PCAP), understand how internal systems resolve domain names, and determine whether the recorded activity contains patterns that warrant further investigation.

The initial packet capture does not contain the DNS activity required for the investigation. You must validate the available evidence, generate controlled DNS queries, and capture the resulting traffic before analyzing it with Zeek. During analysis, a UDP checksum warning may affect the generation of DNS logs, requiring you to investigate the warning and compare the results obtained with and without checksum validation.

### Investigation Tasks

- **Capture validation:** Verify the PCAP format and inspect its contents to confirm that DNS packets are present before running Zeek.
- **DNS traffic collection:** Generate controlled DNS queries and capture the resulting network traffic for offline analysis.
- **DNS transaction analysis:** Examine `dns.log` to identify queried domains, query types, response codes, and returned IP addresses.
- **Pattern identification:** Summarize query frequencies and response codes to identify repeated queries or unsuccessful resolutions that may require further investigation.
- **Checksum troubleshooting:** Investigate Zeek's UDP checksum warning and determine whether checksum-tolerant analysis changes the generated logs.
- **Evidence correlation:** Compare DNS records with connection metadata and investigate whether the recorded source and destination addresses accurately represent the observed communication path.

### Expected Outcome

By completing the investigation, you should be able to analyze DNS transactions using Zeek, explain the significance of the recorded response codes, and document the impact of checksum handling on log generation.

Your final assessment must distinguish observed facts from potential anomalies and unresolved questions. A successful DNS response or an unusual endpoint field does not independently establish malicious activity. Any conclusions about suspicious behavior must be supported by the available packet and log evidence, with limitations clearly documented.


## Investigation Workflow

1. Prepare the lab workspace.
2. Validate the original packet capture and identify whether it contains DNS traffic.
3. Generate DNS queries using `dig`.
4. Capture DNS packets using `tcpdump`.
5. Validate the new PCAP with `tcpdump`.
6. Run Zeek against the capture.
7. Investigate the checksum warning.
8. Reprocess the capture with `-C`.
9. Extract and summarize DNS fields.
10. Record findings and evidence limitations.

## Observed DNS Activity

The successful checksum-tolerant analysis produced three DNS records.

| Domain | Query type | Response code | Recorded answers |
|---|---|---|---|
| `example.com` | A | NOERROR | `172.66.147.243`, `104.20.23.154` |
| `iana.org` | A | NOERROR | `192.0.43.8` |
| `example.net` | A | NOERROR | `104.20.21.8`, `172.66.175.59` |

All three records contain A queries and a `NOERROR` response code. Each domain appears once in the extracted records.

These observations are consistent with successful responses to the controlled DNS test queries. They do not establish malicious activity.

## Key Technical Findings

- The original `quickstart.pcap` was identical to the file initially named `dns-investigation.pcap`, as confirmed by matching SHA-256 hashes.
- The original capture showed HTTP traffic rather than DNS transactions.
- `dns-test.pcap` was captured using `tcpdump` on the WSL `any` interface.
- The resulting PCAP uses Linux cooked capture format, identified as Linux cooked v2.
- Zeek reported a suspected UDP checksum issue during standard analysis.
- Standard analysis did not produce `dns.log`.
- Reprocessing with `zeek -C` generated `dns.log` containing three DNS records.
- The DNS records showed `10.255.255.254` for both `id.orig_h` and `id.resp_h`. This unusual attribution requires further packet-level examination before identifying the actual client and resolver path.

## Important Evidence Limitations

The capture is small and represents controlled test activity rather than a real incident.

The `-C` option allows Zeek to process packets without rejecting them because of checksum errors. It does not repair packet data or establish that all captured packets are valid.

The matching origin and responder addresses in `dns.log` should not be interpreted as proof that a host queried itself. The capture interface, Linux cooked headers, and WSL DNS forwarding behavior may affect how the packet path is represented. The raw packet details should be reviewed before making an attribution claim.

No conclusion about DNS tunneling, malware, command-and-control activity, or domain-generation algorithms is supported by the three DNS records alone.

