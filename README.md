# Wireshark SOC Labs

**Networkwalks Cybersecurity Internship | Wireshark SOC Lab series**
**Intern:** Sheila Chepkoech

## Objective

Work as a SOC analyst: capture live traffic in Wireshark, filter it, and use Follow TCP Stream to find attacks and sensitive data sent in plain text.

## Results

| Lab | Topic | Date | Score |
|-----|-------|------|-------|
| Lab 3 | Identify the Attacker's IP | 29 Oct 2026 | 100 / 100 |
| Lab 4 | Follow TCP Stream | 29 Oct 2026 | 90 / 100 |
| Lab 5 | Follow TCP Stream, part 2 | 28 Oct 2026 | 100 / 100 |
| Lab 6 | HTTP plain text credentials | 28 Oct 2026 | 100 / 100 |
| Lab 7 | Brute force catch | 28 Oct 2026 | 80 / 100 |
| Lab 8 | VoIP call replay | 29 Oct 2026 | 100 / 100 |

**Total: 570 / 600 (95%)**

## Wireshark skills used

- Capturing on the active interface and reading the packet count in the status bar
- Display filters such as `tcp.flags.syn == 1`, `http.request` and `frame contains "<!--"`
- Statistics > Conversations to find the busiest hosts and ports
- Follow > TCP Stream to read whole conversations (red = client, blue = server)
- Reading TCP flags (SYN, SYN-ACK, RST-ACK) to tell open ports from closed ones

---

## Lab 3: Identify the Attacker's IP (100 / 100)

![Lab 3 result](screenshots/lab3_result.png)

**Scenario:** A staff member reported unusual traffic to the company web server. I captured the packets to find the attack type and the attacker's IP. The SOC analyst PC (192.168.10.10) captured from a switch SPAN port, and the web server is 192.168.10.20.

| # | Task | Method | Answer |
|---|------|--------|--------|
| 1 | Capture the traffic | Double-click the active **WiFi** interface, then stop | 309 packets |
| 2 | Show only SYN packets | Filter `tcp.flags.syn == 1` | 73 displayed |
| 3 | Find the attacker | One source sends a long block of SYNs from 3.4 s | 192.168.10.66 |
| 4 | Find the target | Destination of that block | 192.168.10.20 (web server) |
| 5 | Source port | Number before the arrow in the Info column | 52311 |
| 6 | Ports probed | Statistics > Conversations > TCP (one row per port) | 40 |
| 7 | Open ports | Server replied `[SYN, ACK]` | 22, 80, 443 |
| 8 | Reply for closed ports | Server replied with a reset | RST, ACK |
| 9 | Closed ports | 40 tried - 3 open | 37 |
| 10 | Attack type | One host, many ports, very fast | Port scan (TCP SYN scan) |

**Why a port scan and not a SYN flood:** every probe used the same source port (52311) and a different destination port, about 12.5 ms apart, which means an automated tool. The attacker wanted the replies to learn which services are open. A flood would hit one port with huge volumes of SYNs.

**What I learned:** one host opening dozens of connections in a row stands out in Statistics > Conversations. A scan is usually the first step before an attack on the open services (here SSH and the web ports), so it should be detected and blocked early, for example with firewall rate limits or IDS rules for SYN scans.

---

## Lab 4: Follow TCP Stream (90 / 100)

![Lab 4 result](screenshots/lab4_result.png)

**Scenario:** Some staff use an old office chat tool that sends messages in plain text. I used Follow TCP Stream to read one chat and see what was shared.

| # | Task | Method | Answer |
|---|------|--------|--------|
| 1 | Capture the traffic | Start and stop a capture | Done |
| 2 | Find the chat port | TCP traffic that is not port 80 or 443 | 5000 (marked wrong, see note) |
| 3 | Who started the chat | Source of the first SYN to port 5000 | 192.168.10.45 |
| 4 | Stream number | Follow > TCP Stream title `tcp.stream eq N` | 5 |
| 5 | Size of the chat | Bottom bar: Entire conversation | 413 bytes |
| 6 | Who needs help | Red (client) text | Ali, from Accounts |
| 7 | The problem drive | Read the chat | Finance drive |
| 8 | Temporary password | Sent by IT (Sara) | Tiger#4821 |
| 9 | Guest Wi-Fi password | Sent by IT | BlueHarbor77 |
| 10 | Text colour | Client text | Red |

**Note on task 2:** The platform marked this wrong. A later question in the same lab states "The chat uses TCP port 5000", and the first SYN in the capture (frame 81) goes to port 5000, so the finding is consistent.

**What I learned:** the chat ("Office Chat v1.2") was fully readable, including a temporary password and the guest Wi-Fi password. Chat tools should use TLS, and passwords should never be shared over chat.

---

## Lab 5: Follow TCP Stream, part 2 (100 / 100)

![Lab 5 result](screenshots/lab5_result.png)

**Scenario:** A staff member browsed the company intranet and an admin thought a secret was left inside a web page. I captured the traffic and used Follow TCP Stream to find it.

| # | Task | Method | Answer |
|---|------|--------|--------|
| 1 | Capture the traffic | Start and stop a capture | Done |
| 2 | Find the pages | Filter `http.request` | Five pages opened by 192.168.10.45: help, index, about, staff, contact |
| 3 | Where is it hidden | Searched with `frame contains "<!--"` | In an HTML comment |
| 4 | Find the stream | `[Stream index]` in the TCP section | 7 |
| 5 | The code word | Inside the comment | Falcon-9921 |
| 6 | The backup server | Inside the comment | 192.168.10.77 |
| 7 | Which direction | Blue text in Follow TCP Stream | Server to client |
| 8 | Status code | Start of the server reply | 200 (OK) |
| 9 | The downloaded file | `GET /download/report.txt` | report.txt |
| 10 | Who made the report | Read the file in Follow TCP Stream | Answered correctly |

**Capture evidence:** my own Wireshark window with the `http.request` filter applied, listing the intranet page requests to the web server (192.168.10.20).

![Lab 5 http.request filter in Wireshark](screenshots/lab5_http_request_filter.png)

The page `/staff.html` ended with: `<!-- IT note: backup server is 192.168.10.77, access code Falcon-9921. Remove before publishing. -->`

**What I learned:** `http contains` is not a valid filter because `http` is a protocol, not a field, so `frame contains "..."` works instead. Browsers hide comments, but they travel over the network in plain text, so secrets must be removed before a page is published and sites should use HTTPS.

---

## Lab 6: HTTP plain text credentials (100 / 100)

![Lab 6 result](screenshots/lab6_result.png)

Tasks: capture the traffic, which protocol, the method, the login page, the PC that signed in, how many attempts, the username, the password, the packet details line, and the fix.

Login details sent over plain HTTP can be read in the capture. HTTPS protects them in transit.

---

## Lab 7: Brute force catch (80 / 100)

![Lab 7 result](screenshots/lab7_result.png)

Tasks: capture the traffic, the protocol, the target, the attacker, the username, failure code, failed logins, success code, the password that worked, and the attack type. Tasks 2 (protocol) and 7 (failed logins) were marked wrong.

A brute force attack shows up as many failed login replies from one source against one target, followed by a success.

---

## Lab 8: VoIP call replay (100 / 100)

![Lab 8 result](screenshots/lab8_result.png)

Tasks: capture the traffic, call setup protocol, voice protocol, caller extension, caller name, the codec, who answered, card number, expiry date, and the fix.

Unencrypted VoIP calls can be replayed from a capture, including sensitive details spoken on the call. Encrypting call signalling and audio is the fix.
