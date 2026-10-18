# Zeek DNS Investigation Lab

## Overview

This project documents a hands-on DNS investigation using Zeek 8.0.10 and an offline packet capture in Ubuntu 24.04 running under WSL.

The investigation began with a packet capture that did not contain DNS transactions. A separate capture was then created using `tcpdump` while DNS queries were generated with `dig`. During analysis, Zeek reported a possible UDP checksum issue and did not initially generate `dns.log`. Reprocessing the capture with checksum validation disabled using the `-C` option successfully generated the DNS log.

The lab demonstrates practical DNS log analysis, packet capture validation, checksum troubleshooting, and evidence-based investigation documentation.

## Objectives

- Analyze DNS transactions in an offline PCAP using Zeek.
- Extract queried domains, DNS query types, response codes, and returned answers.
- Summarize query patterns and response-code distributions.
- Investigate Zeek's UDP checksum warning and compare analysis results.
- Examine connection metadata and identify limitations in endpoint attribution.
- Document findings, troubleshooting steps, and evidence limitations without making unsupported security claims.

## Lab Environment

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

A SOC analyst needs to inspect DNS activity from a small packet capture and determine what can be established from the recorded transactions.

The initial capture, `quickstart.pcap`, contained HTTP traffic to external IP addresses on port 80. It did not produce `dns.log`, so it was unsuitable for the intended DNS investigation.

A new capture, `dns-test.pcap`, was created while DNS queries were sent to the WSL-configured resolver at `10.255.255.254`. The capture was analyzed with Zeek, which reported suspected invalid UDP checksums. The standard analysis did not generate `dns.log`, but analysis with `-C` did.

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

## Conclusion

This lab demonstrates how to validate packet captures, generate controlled DNS traffic, troubleshoot Zeek checksum warnings, and analyze DNS records using Zeek's command-line tools.

The final result is a successfully generated `dns.log` with three observed DNS transactions. The investigation also highlights why packet-level verification and explicit documentation of limitations are essential when interpreting network telemetry.

## Skills Demonstrated

- Zeek offline PCAP analysis
- DNS query and response investigation
- `tcpdump` packet capture and validation
- DNS testing with `dig`
- Zeek checksum troubleshooting
- Log extraction using `zeek-cut`
- Evidence correlation and technical documentation
- Evidence-based SOC investigation
