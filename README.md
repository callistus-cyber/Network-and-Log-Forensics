# Network and Log Forensics: Identifying a Rogue User

> A SOC-style investigation that traces unauthorized email activity at a fictional company ("Boring Office") back to a specific individual by pivoting across packet capture, DHCP logs, and Windows security logs.

**Author:** Callistus : Cyber Security Analyst, GRC, Security Architect

**Discipline:** Blue Team / Digital Forensics & Incident Response
**Tools:** Wireshark · DHCP server logs · Windows Security Event Logs

---

## TL;DR

A user inside Boring Office sent sensitive emails while impersonating another employee. Using only a `.pcap`, the DHCP server log, and the workstation security log, I pivoted from **email payload → source IP → host device → logged-in user** and identified the rogue actor as **John Doe**, working from host **USER2** at **12:11 PM** the same day the emails were sent.

---

## Objective

Simulate a real-world Security Operations Center (SOC) investigation. Analyze network activity and host logs to identify a rogue user inside the fictional company "Boring Office" — an individual suspected of impersonating an employee and exfiltrating sensitive information over email.

The project demonstrates the critical thinking, technical skills, and forensic methodology used to resolve security incidents, and reinforces how layered log analysis and packet inspection are used together in modern blue-team operations.

## Project Goals

- **Analyze network traffic** : Investigate a provided `.pcap` file to extract key network activity details, including the source IP of the suspect email session.
- **Leverage DHCP logs** : Use DHCP lease records to correlate the identified IP address back to a specific host device on the network.
- **Investigate security logs** : Review Windows account security logs to determine which user account was actively logged in to that host device, surfacing the rogue user's identity.

## Skills Demonstrated

- **Packet analysis** : Using Wireshark to filter `.pcap` files, follow protocol-specific traffic, and extract attribution-grade artifacts (IP addresses, sender fields, suspicious patterns).
- **Log correlation** : Cross-referencing DHCP lease activity against network traffic to map an IP address to a physical or virtual host within an environment.
- **User attribution via security logs** : Reading logon/logoff events to attribute host activity to a specific user account.
- **Incident investigation workflow** : Practicing the structured Blue Team approach: evidence collection → analysis → attribution → conclusion.

---

## Step 1 — Identify the Source IP from the PCAP

![Wireshark — initial packet capture](https://github.com/user-attachments/assets/e9ea9585-8b24-404a-8023-e8d576d2aceb)

**1. Initial packet count.** Open the `.pcap` and observe the total packet count (e.g., 60 packets). This represents all captured network traffic in scope.

**2. First filter ; narrow to email traffic.** Apply a Wireshark display filter to focus on email-related traffic:
smtp

This narrows the view to **Simple Mail Transfer Protocol (SMTP)** packets only, since the suspected activity involves outbound email.

![Wireshark filtered to SMTP traffic](https://github.com/user-attachments/assets/6f2cec54-ec7f-43de-b88c-9d045428ccf8)

**3. Refine ; locate the sender field.** Tighten the filter to surface only the SMTP packet(s) carrying a `FROM:` header (the sender's email address):
smtp contains "FROM"

![Wireshark filtered to SMTP packets containing FROM](https://github.com/user-attachments/assets/9020a470-98b0-4617-8521-2844bff744a6)

**4. Inspect the remaining packet.** A single packet remains. The Info column confirms it's tied to the impersonated user's email account.

**5. Extract the source IP.** From the packet details, the source IP is:

> **Source IP: `10.10.1.4`**

This IP is now the lead. Every subsequent step pivots from here.

---

## Step 2 : Correlate the IP to a Host Device (DHCP Logs)

**1. Open the DHCP log.** Review the log structure and focus on lease activity around the time the emails were sent (~12:50 PM).

![DHCP log entries](https://github.com/user-attachments/assets/e11099b5-2da3-4812-b86a-9cda3cac251b)

**2. Narrow the timeframe.** The rogue user had to obtain a lease *before* sending the emails, so the relevant lease event should appear earlier in the day; in this case, around **12:11 PM**.

**3. Locate the matching lease event.** At **12:11:27 PM**, the DHCP server assigned `10.10.1.4` to a specific host.

**4. Identify the host device.** The lease record ties `10.10.1.4` to host **`USER2`**.

> **Host device: `USER2`**

With the host identified, the next pivot is into that workstation's security log to learn *who* was logged in.

---

## Step 3 : Attribute the Activity to a User (Security Logs)

**1. Open the security log** for the time window covering the suspect activity.

![Security log — logon and logoff events](https://github.com/user-attachments/assets/c971c8f5-ecbc-4543-82b5-7de0af61a4b2)

**2. Walk the logon/logoff events.** Note that the first two entries are tied to host **`USER1`** (logged-in user: **Jane Doe**) — these are unrelated to our IP of interest.

**3. Focus on `USER2` entries.** The remaining entries cover the logon/logoff session for host **`USER2`** — the device that received the `10.10.1.4` lease.

**4. Confirm the rogue user.** The `USER2` session is owned by **John Doe**, and the session window cleanly overlaps with the time the sensitive emails were sent.

> **Rogue user: John Doe** (logged in to host `USER2`, leased `10.10.1.4`, source of the SMTP `FROM` packet at ~12:50 PM)

---

## Investigation Chain of Custody

| Pivot | Source | Artifact | Result |
|---|---|---|---|
| 1. Email content → source IP | Wireshark `.pcap` | SMTP packet with `FROM:` header | `10.10.1.4` |
| 2. IP → host device | DHCP server log | Lease event at 12:11:27 PM | Host `USER2` |
| 3. Host → user identity | Windows security log | Logon session for `USER2` | **John Doe** |

---

## Conclusion

By layering three independent data sources ; packet capture, DHCP lease activity, and host security logs. The investigation produced a defensible attribution chain from the email payload all the way to a named individual. **John Doe**, working from host **USER2**, is the rogue user responsible for impersonating a colleague and sending the sensitive emails.

The exercise reinforces a core SOC discipline: a single log source rarely tells the whole story, but a small handful of well-correlated sources almost always does.

## What I'd Do Differently in a Real Engagement

- **Preserve evidence integrity first** : Hash the original `.pcap` and log exports, work from copies, and document every pivot in a case management system.
- **Expand the timeline** : Pull email server logs, EDR telemetry, and proxy logs to corroborate the SMTP packet and rule out shared-account or stolen-credential scenarios.
- **Validate the chain of custody** : Confirm DHCP lease uniqueness for the period in question and verify there were no concurrent sessions on `USER2`.
- **Hand off cleanly** : Package findings into an incident report with timestamps in UTC, screenshots of the filtered evidence, and a recommended response (account reset, manager notification, HR / legal escalation per policy).


---

*Project repository:* `Network-and-Log-Forensics`
*Author:* Callistus — Cyber Security Analyst, GRC, Security Architect
      
