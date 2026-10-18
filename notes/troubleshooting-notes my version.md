# Troubleshooting Notes

## 1. Issue: DNS Log Not Generated from the Original PCAP

### Symptoms

Running Zeek against `dns-investigation.pcap` generated:

- `conn.log`
- `files.log`
- `http.log`
- `packet_filter.log`
- `weird.log`

However, `dns.log` was absent.

### Investigation

The file was checked with:

    ls -lh dns-investigation.pcap
    file dns-investigation.pcap
    sha256sum dns-investigation.pcap

The hash was then compared with the original Lab 01 capture:

    sha256sum ~/zeek-labs/lab01/quickstart.pcap \
              ~/zeek-labs/lab03/dns-investigation.pcap

Both files had the same SHA-256 hash:

    090244c27e3c2ab6f4d9f63a11076ae9bd887ee67b28bfd9d1e62293065318a8

The file was also inspected using:

    sudo tcpdump -nn -r dns-investigation.pcap

The output showed TCP connections to external addresses on port 80, HTTP requests, and HTTP 301 responses.

### Resolution

The original capture was unsuitable for the intended DNS investigation. A separate capture was created rather than attempting to generate DNS findings from HTTP-only evidence.

### Lesson Learned

A filename does not determine the contents of a PCAP. Always validate the capture and verify the presence of the intended protocol before beginning an investigation.

## 2. Issue: No Alternative PCAP Found

### Symptoms

Searching the provided cybersecurity resources folder returned only `quickstart.pcap`.

### Diagnostic Command

    find "/mnt/d/important docs/Cybersecurity labs and resources" \
      -type f \( -iname "*.pcap" -o -iname "*.pcapng" \) \
      -printf '%s bytes  %p\n' | sort -nr

### Result

Only one capture was found in the specified folder.

### Resolution

A controlled DNS capture was generated using `tcpdump` while DNS queries were sent with `dig`.

## 3. Issue: Capture File Was Truncated

### Symptoms

Both `tcpdump` and Zeek reported:

    truncated dump file; tried to read 4 file header bytes, only got 0

### Explanation

The capture file did not contain a complete readable PCAP header. This is consistent with an empty or incompletely written capture.

### Resolution

The broken capture was removed and a fresh capture was created. The capture process was allowed to run while DNS queries were generated and was stopped with `Ctrl+C` to finalize the file cleanly.

### Lesson Learned

Before running Zeek, validate the PCAP with `ls -lh`, `file`, and packet inspection. A filename alone is not evidence that the capture is valid.

## 4. Issue: DNS Capture Generated but Standard Zeek Analysis Did Not Produce dns.log

### Capture Command

    sudo tcpdump -i any -nn -s 0 -U \
      -w dns-test.pcap 'udp port 53 or tcp port 53'

### DNS Queries

    dig @10.255.255.254 example.com
    dig @10.255.255.254 iana.org
    dig @10.255.255.254 example.net

### Capture Properties

The captured file was identified as:

    pcap capture file, microsecond ts (little-endian) - version 2.4 (Linux cooked v2, capture length 262144)

The capture command reported six packets captured, twelve packets received by filter, and zero packets dropped by the kernel.

### Zeek Warning

Standard analysis produced a warning from `find-checksum-offloading.zeek` indicating that the trace likely contained invalid UDP checksums. Zeek explained that packets with invalid checksums may be discarded unless checksum validation is disabled.

### Resolution

The capture was reprocessed using `-C`:

    /opt/zeek/bin/zeek -C -r ../dns-test.pcap

This generated `dns.log`.

### Interpretation

The result shows that checksum-tolerant processing allowed Zeek to produce the DNS log. It does not prove that every checksum was invalid or that checksum validation was the only factor affecting the initial output.

The `-C` option bypasses checksum validation during analysis; it does not repair packet contents.

## 5. Issue: Origin and Responder Addresses Match

### Observation

The extracted DNS records show:

    id.orig_h: 10.255.255.254
    id.resp_h: 10.255.255.254

This occurs for all three records.

### Why It Matters

The address matches the configured WSL DNS resolver, but the identical origin and responder fields are unusual and make endpoint attribution uncertain.

The capture was taken on the WSL `any` interface and uses Linux cooked v2 link-layer framing. WSL DNS forwarding and virtualized networking may influence the packet representation observed by Zeek.

### Recommended Diagnostics

Inspect the captured DNS packets:

    sudo tcpdump -nn -vvv -r ~/zeek-labs/lab03/dns-test.pcap

Inspect Zeek connection metadata:

    cd ~/zeek-labs/lab03/zeek-output-dns-checksum

    /opt/zeek/bin/zeek-cut uid id.orig_h id.orig_p id.resp_h id.resp_p proto service conn_state < conn.log

Compare the connection metadata with the DNS records. Do not assume the true client address from the DNS log alone.

### Status

Unresolved. Further packet-level analysis is required before explaining the matching addresses.

## 6. Useful Diagnostic Commands

### Validate the capture

    ls -lh dns-test.pcap
    file dns-test.pcap
    sha256sum dns-test.pcap

### Inspect DNS packets

    sudo tcpdump -nn -r dns-test.pcap 'port 53'

### Run standard Zeek analysis

    /opt/zeek/bin/zeek -r dns-test.pcap

### Run checksum-tolerant Zeek analysis

Run this from a separate output directory:

    /opt/zeek/bin/zeek -C -r ../dns-test.pcap

### List generated logs

    ls -lh *.log

### Extract DNS fields

    /opt/zeek/bin/zeek-cut ts uid id.orig_h id.resp_h query qtype_name rcode_name answers < dns.log

### Count DNS transactions

    grep -vc '^#' dns.log

### Summarize query types

    /opt/zeek/bin/zeek-cut qtype_name < dns.log | sort | uniq -c | sort -nr

### Summarize response codes

    /opt/zeek/bin/zeek-cut rcode_name < dns.log | sort | uniq -c | sort -nr

### Summarize queried domains

    /opt/zeek/bin/zeek-cut query < dns.log | sort | uniq -c | sort -nr

