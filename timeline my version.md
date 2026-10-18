# Investigation Timeline 
## Timeline

### 1. Lab Workspace Created

**Action:** Created the Lab 03 working directory.

**Command:**

    mkdir -p ~/zeek-labs/lab03

**Outcome:** The working directory was available for the DNS investigation.

### 2. Initial Capture Analyzed

**Action:** Ran Zeek against `dns-investigation.pcap`.

**Outcome:** Zeek generated connection, HTTP, file, packet-filter, and weird logs, but no DNS log.

**Evidence:** Generated log listing from the initial Zeek run.

### 3. Capture Identity Verified

**Action:** Compared the SHA-256 hashes of `quickstart.pcap` and `dns-investigation.pcap`.

**Outcome:** Both files had the same hash:

    090244c27e3c2ab6f4d9f63a11076ae9bd887ee67b28bfd9d1e62293065318a8

**Finding:** The Lab 03 input was identical to the Lab 01 capture.

### 4. Original PCAP Inspected

**Action:** Read the original capture with `tcpdump`.

**Outcome:** The displayed packets showed TCP connections to external IP addresses on port 80, HTTP requests, and HTTP 301 responses.

**Finding:** The original capture did not provide the required DNS evidence.

### 5. Alternative Capture Search Completed

**Action:** Searched the specified cybersecurity resources folder for PCAP and PCAPNG files.

**Outcome:** Only `quickstart.pcap` was found.

**Decision:** Generate a controlled DNS capture rather than reuse the HTTP-only dataset.

### 6. DNS Test Capture Created

**Action:** Started `tcpdump` on the WSL `any` interface and generated DNS queries using `dig`.

**Queries generated:**

- `example.com`
- `iana.org`
- `example.net`

**Capture file:** `dns-test.pcap`

**Outcome:** A new PCAP was created. The capture command reported six packets captured, twelve packets received by filter, and zero packets dropped by the kernel.

### 7. Capture Format Verified

**Action:** Inspected `dns-test.pcap` with `file`.

**Outcome:** The file was identified as a PCAP 2.4 capture using Linux cooked v2 link-layer framing.

**Finding:** The capture was structurally recognized as a packet capture. Packet-level inspection was still necessary to confirm its DNS contents.

### 8. Standard Zeek Analysis Attempted

**Action:** Ran Zeek against `dns-test.pcap` without additional options.

**Outcome:** Zeek warned that the trace likely contained invalid UDP checksums. The standard run generated `conn.log`, `packet_filter.log`, and `reporter.log`, but did not generate `dns.log`.

**Finding:** Checksum validation was a potential factor affecting DNS analysis.

### 9. Checksum-Tolerant Analysis Performed

**Action:** Ran Zeek with the `-C` option in a separate output directory.

**Command:**

    /opt/zeek/bin/zeek -C -r ../dns-test.pcap

**Outcome:** Zeek generated `dns.log`, `conn.log`, and `packet_filter.log`.

**Finding:** Checksum-tolerant processing produced the DNS log. The original PCAP remained unchanged.

### 10. DNS Transactions Extracted

**Action:** Extracted timestamps, UIDs, endpoint fields, queried domains, query types, response codes, and answers from `dns.log`.

**Outcome:** Three DNS transactions were observed.

| Domain | Query type | Response code | Answers |
|---|---|---|---|
| `example.com` | A | NOERROR | `172.66.147.243`, `104.20.23.154` |
| `iana.org` | A | NOERROR | `192.0.43.8` |
| `example.net` | A | NOERROR | `104.20.21.8`, `172.66.175.59` |

### 11. Query and Response Summaries Reviewed

**Action:** Counted query types, response codes, and queried domains.

**Outcome:**

- A queries: 3
- NOERROR responses: 3
- Each queried domain appeared once in the extracted records.

**Finding:** The three recorded transactions contained successful DNS responses, with no repeated domain among this small sample.

### 12. Endpoint Attribution Limitation Identified

**Action:** Reviewed the DNS endpoint fields.

**Outcome:** Both `id.orig_h` and `id.resp_h` showed `10.255.255.254` for all three transactions.

**Finding:** The actual client-to-resolver path could not be established from the extracted fields alone.

**Status:** Requires further packet-level analysis.

## Final Status

**DNS log generation:** Successful with `-C`  
**DNS transactions extracted:** 3  
**Observed response codes:** All `NOERROR`  
**Checksum warning:** Documented  
**Endpoint attribution:** Unresolved  
**Malicious activity established:** No

