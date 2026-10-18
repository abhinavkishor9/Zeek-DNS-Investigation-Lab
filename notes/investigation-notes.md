# Investigation Notes — Zeek DNS Investigation

## 1. Case Overview

**Investigation:** DNS traffic analysis using Zeek  
**Lab:** Zeek Lab 03  
**Platform:** Ubuntu 24.04 on WSL  
**Zeek version:** 8.0.10  
**Primary evidence:** `dns-test.pcap`  
**Primary log:** `dns.log`  
**Analysis method:** Offline PCAP processing

### Investigation Objective

Analyze DNS transactions from a controlled packet capture, identify the queried domains and their responses, investigate the effect of checksum validation on Zeek's output, and document what the available evidence establishes.

## 2. Investigation Background

The investigation initially used `quickstart.pcap`, which had previously been analyzed in Zeek Lab 01.

The file was copied into the Lab 03 directory and renamed `dns-investigation.pcap`. Comparing the SHA-256 hashes confirmed that the two files were identical.

The capture contained TCP connections to external IP addresses on port 80 and HTTP responses with status code 301. It did not contain the intended DNS transactions.

A separate capture was then created using `tcpdump` while DNS queries were generated with `dig`.

## 3. Evidence Reviewed

| Evidence | Purpose | Result |
|---|---|---|
| `quickstart.pcap` | Initial capture review | HTTP traffic observed |
| `dns-investigation.pcap` | Initial Lab 03 input | Identical to `quickstart.pcap` |
| `dns-test.pcap` | Controlled DNS capture | Captured using `tcpdump` |
| `conn.log` | Connection metadata | Generated during checksum-tolerant analysis |
| `dns.log` | DNS transaction analysis | Three records generated with `-C` |
| `packet_filter.log` | Packet filtering metadata | Generated |
| Zeek warning output | Checksum troubleshooting | Suspected invalid UDP checksums |

## 4. Capture and Analysis Method

The controlled DNS capture was created using the following command:

    sudo tcpdump -i any -nn -s 0 -U -w dns-test.pcap 'udp port 53 or tcp port 53'

DNS queries were generated using:

    dig @10.255.255.254 example.com
    dig @10.255.255.254 iana.org
    dig @10.255.255.254 example.net

The resulting PCAP was inspected with `tcpdump` before being processed by Zeek.

The initial Zeek analysis was run with:

    /opt/zeek/bin/zeek -r ../dns-test.pcap

Zeek reported a warning that the trace likely contained invalid UDP checksums. The standard run generated connection and other logs but did not generate `dns.log`.

The capture was then reprocessed with checksum validation disabled:

    /opt/zeek/bin/zeek -C -r ../dns-test.pcap

This run generated `dns.log`.

## 5. DNS Evidence

The following records were extracted from `dns.log` using `zeek-cut`:

    /opt/zeek/bin/zeek-cut ts id.orig_h id.resp_h query qtype_name rcode_name answers < dns.log

### Observed Transactions

| Query | Type | Response code | Answers |
|---|---|---|---|
| `example.com` | A | NOERROR | `172.66.147.243`, `104.20.23.154` |
| `iana.org` | A | NOERROR | `192.0.43.8` |
| `example.net` | A | NOERROR | `104.20.21.8`, `172.66.175.59` |

### Query Type Distribution

The command:

    /opt/zeek/bin/zeek-cut qtype_name < dns.log | sort | uniq -c | sort -nr

returned:

    3 A

All three records were A queries, which request IPv4 address records.

### Response Code Distribution

The command:

    /opt/zeek/bin/zeek-cut rcode_name < dns.log | sort | uniq -c | sort -nr

returned:

    3 NOERROR

All three records contained `NOERROR`.

### Query Frequency

The command:

    /opt/zeek/bin/zeek-cut query < dns.log | sort | uniq -c | sort -nr

returned one occurrence each for:

- `example.com`
- `iana.org`
- `example.net`

No repeated domain queries were observed in these three DNS records.

## 6. DNS Endpoint Attribution

The extracted DNS records showed the following values:

- `id.orig_h`: `10.255.255.254`
- `id.resp_h`: `10.255.255.254`

Both fields contained the same address for all three records.

This is an attribution limitation that requires further investigation. The address is configured as the WSL DNS resolver, but the matching values do not independently establish the original client address or prove that the DNS request was sent from the resolver to itself.

The capture was made on the WSL `any` interface and was saved in Linux cooked v2 format. WSL networking and DNS forwarding may affect the visible packet path.

Further review of the captured packet headers, DNS message contents, and connection records is needed before making a definitive statement about the client-to-resolver path.

## 7. Checksum Warning

During standard analysis, Zeek displayed a warning that the trace likely contained invalid UDP checksums, possibly due to NIC checksum offloading.

Zeek noted that packets with invalid checksums may be discarded unless checksum validation is disabled.

Reprocessing with `-C` resulted in a generated `dns.log` containing three transactions.

### Interpretation

The different outputs indicate that checksum handling affected the analysis result. However, this alone does not prove that checksum errors were the only possible cause of the missing log, nor does it establish whether the original checksums were invalid on the network or affected by the capture path.

The original PCAP was not modified by the `-C` option.

## 8. Findings

### Finding 1 — DNS transactions successfully analyzed

Three DNS transactions were recorded after running Zeek with `-C`.

**Evidence:** `dns.log` contains queries for `example.com`, `iana.org`, and `example.net`.

### Finding 2 — All recorded queries returned NOERROR

Each extracted DNS record contains the response code `NOERROR`.

**Evidence:** The response-code summary contains three `NOERROR` entries.

### Finding 3 — All observed queries requested A records

The query-type summary contains three A records.

**Evidence:** The query-type distribution returned `3 A`.

### Finding 4 — No repeated queried domains were observed

Each of the three domains appears once in the extracted records.

**Evidence:** The query-frequency summary contains one occurrence for each domain.

### Finding 5 — Checksum handling affected log generation

The standard Zeek run warned about suspected invalid UDP checksums and did not produce `dns.log`. The run using `-C` produced the DNS log.

**Evidence:** Terminal output from both analysis runs.

### Finding 6 — Endpoint attribution remains unresolved

The origin and responder fields contain the same IP address for each DNS record.

**Evidence:** Extracted `dns.log` fields.

**Limitation:** The current evidence does not establish the exact client-to-resolver packet path.

## 9. Assessment

The three recorded transactions are consistent with the controlled DNS queries generated for this lab. Their response codes indicate successful DNS responses, and the records contain returned IPv4 addresses.

No evidence in these records establishes malicious activity. The small sample size also prevents meaningful conclusions about broader DNS behavior.

The checksum warning and endpoint attribution issue remain technical limitations to document and investigate further.

## 10. Recommended Follow-Up

- Inspect the packet-level contents of `dns-test.pcap`.
- Review the full `conn.log`, including ports, protocols, and connection identifiers.
- Compare the DNS transaction UIDs with connection metadata where matching records exist.
- Determine why Zeek recorded the same origin and responder address.
- Preserve the original PCAP and record the exact Zeek commands used.
- If necessary, capture traffic through a suitable interface or use a known-good DNS PCAP to validate the expected logging behavior.

## 11. Final Conclusion

Zeek successfully generated DNS transaction records when the capture was analyzed with checksum validation disabled. Three DNS queries were observed, all using A records and returning `NOERROR`.

The available evidence supports documenting the observed DNS responses and the checksum-related analysis behavior. It does not support classifying the activity as malicious or definitively attributing the DNS transactions to a specific client-to-resolver path.
