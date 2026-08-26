# Wireshark PCAP Investigation

## 1. Main Host

### Observation

I used the following command to obtain endpoint statistics from the PCAP:

`tshark -r 2020-09-25-traffic-analysis-exercise.pcap -q -z endpoints,ip`

The internal IP that immediately stood out was `10.0.0.179`, which had significantly higher total traffic, particularly in terms of bytes, than the other internal endpoints.

### Reasoning

`10.0.0.179` is an internal address because it belongs to the private `10.0.0.0/8` address range. Its significantly higher traffic volume made it the primary host of interest for the remainder of the investigation.

At this stage, this does not by itself prove that the host was compromised, but it provided a reasonable starting point for further investigation.

### Evidence

../screenshots/2026-08-25_14-21.png

## 2. Who Does the Host Talk To?

### Observation

I used the following command to examine IP conversations within the PCAP:

`tshark -r 2020-09-25-traffic-analysis-exercise.pcap -q -z conv,ip`

The main external IP addresses associated with significant traffic involving `10.0.0.179` were:

* `185.61.152.63`
* `37.120.174.218`
* `104.87.12.44`
* `198.12.66.108`
* `13.107.42.23`

### Interesting Connections

* `10.0.0.179 <-> 185.61.152.63` — the host transmitted approximately 354 kB while receiving approximately 21 kB. The large outbound imbalance made this connection particularly interesting.
* `10.0.0.179 <-> 37.120.174.218` — the host received approximately 682 kB while transmitting only 7,483 bytes.
* Similar traffic imbalances were also visible for:

  * `104.87.12.44`
  * `198.12.66.108`
  * `13.107.42.23`

### Reasoning

The traffic patterns suggested that `10.0.0.179` was receiving significant amounts of data from several external hosts, while the connection with `185.61.152.63` showed the opposite pattern, with substantially more data being transmitted from the internal host.

The relative start times were also interesting. Traffic received from three of the external IPs appeared to occur before the later large outbound transfer towards `185.61.152.63`.

At this point, however, the contents and purpose of the traffic were unknown. I therefore decided to investigate DNS to establish which domains were associated with the external IPs and determine whether the connections appeared related.

### Evidence

![Interesting connection](screenshots/2026-08-25_22-11.png)

## 3. DNS Analysis

### Observation

I used the following command to examine DNS traffic:

`tshark -r 2020-09-25-traffic-analysis-exercise.pcap -Y dns`

I then focused specifically on the five external IP addresses identified during the previous stage.

The following domain associations were identified:

* `10.0.0.179` requested `mail.big3.icu` → `185.61.152.63`
* `10.0.0.179` requested `paste.nrecom.net` → `37.120.174.218`
* `10.0.0.179` requested `dlassets-ssl.xboxlive.com` → `104.87.12.44`
* `10.0.0.179` requested `config.edge.skype.com` → `13.107.42.23`

No corresponding DNS request for `198.12.66.108` was observed in the captured DNS traffic.

### Reasoning

The DNS investigation provided useful context for the external IP addresses.

Of the five IPs I was initially interested in, four could be associated with domains within the captured DNS traffic. The domains also provided additional context for the connections, with some appearing to correspond to legitimate services rather than immediately indicating malicious infrastructure.

The absence of a DNS request for `198.12.66.108` was also noted, although this does not necessarily mean that the host did not communicate with the IP because DNS activity may have occurred outside the captured traffic or the address may have been obtained through another mechanism.

The DNS analysis did not introduce any additional evidence strong enough to change my investigation at this stage, so I moved on to HTTP traffic.

### Evidence

![DNS](screenshots/2026-08-26_00-50.png)

![DNS](screenshots/2026-08-26_00-50_1.png)

![DNS](screenshots/2026-08-26_00-50_2.png)

![DNS](screenshots/2026-08-26_00-53.png)

## 4. HTTP Analysis

### Observation

I used:

`tshark -r 2020-09-25-traffic-analysis-exercise.pcap -Y http`

The HTTP traffic revealed a particularly interesting request from `10.0.0.179` to `198.12.66.108`.

The host made a `GET` request for:

`/jojo.exe`

The server subsequently returned an `HTTP/1.1 200 OK` response and transferred the requested executable to `10.0.0.179`.

### Reasoning

The `.exe` file immediately raised my suspicion because it represents an executable program being downloaded onto the host.

This finding was particularly significant when considered alongside the earlier traffic analysis. The host had previously received substantial traffic from several external IPs, and later transmitted a significantly larger amount of data towards `185.61.152.63`.

At this stage, I hypothesised that the executable could have been involved in subsequent activity on the host, potentially including information collection or data extraction. However, the HTTP evidence alone was not sufficient to determine what the executable actually did.

I therefore used the graphical Wireshark application to investigate the relevant traffic in greater detail.

### Evidence

![HTTP traffic](screenshots/2026-08-26_01-05.png)

![HTTP traffic](screenshots/2026-08-26_01-20.png)

## 5. TLS Analysis

### Observation

I examined the TLS traffic using:

`tshark -r 2020-09-25-traffic-analysis-exercise.pcap -Y tls`

The TLS traffic did not provide definitive evidence supporting my current hypothesis.

I also investigated the previously flagged IP addresses individually to determine whether their traffic showed characteristics that could indicate command-and-control activity.

### Reasoning

After investigating the traffic associated with the following IPs:

* `37.120.174.218`
* `104.87.12.44`
* `13.107.42.23`

I decided to reduce their priority.

I could not identify clear timing patterns or other evidence that strongly suggested command-and-control activity. The traffic also appeared consistent with encrypted communications, meaning the contents could not be directly assessed from the captured payload.

In contrast, I maintained my suspicion of `185.61.152.63`.

This IP was associated with `mail.big3.icu` and was involved in SMTP traffic. It also had the largest significant outbound transfer from `10.0.0.179` identified during the earlier investigation.

The timing was also notable because this SMTP activity occurred after the earlier HTTP download of `/jojo.exe` from `198.12.66.108`.

At this point, my working hypothesis became that the executable may have been involved in collecting information from the host before transmitting it to `185.61.152.63`.

I therefore decided to investigate the SMTP traffic directly using Wireshark.

## 6. Wireshark Investigation

### Investigation 1 — SMTP Traffic

I investigated the SMTP traffic between `10.0.0.179` and `185.61.152.63` using Wireshark.

The TCP stream provided evidence that connected the SMTP activity to the earlier findings. The stream contained references to the email address:

`jojo@big3.icu`

The internal host subsequently transmitted system and hardware information to the external server.

Following this information, a large block of data was also transmitted in what appeared to be an attachment.

This was significant because it provided evidence of information being collected from the compromised host and subsequently transmitted to an external server.

### Evidence

![WIRESHARK traffic](screenshots/2026-08-26_01-43.png)

## 7. Conclusion

The investigation identified `10.0.0.179` as the primary host of interest due to its significantly higher traffic volume compared with the other internal endpoints.

Initial investigation identified five external IP addresses that warranted further analysis. DNS analysis associated four of these with hostnames, while no corresponding DNS request was observed for `198.12.66.108`.

Further investigation reduced the suspicion surrounding several of the initial connections after no clear evidence of malicious behaviour or command-and-control activity was identified.

The most significant finding was the HTTP request from `10.0.0.179` to `198.12.66.108` for `/jojo.exe`, followed by a successful `HTTP/1.1 200 OK` response and transfer of the executable to the host.

Subsequent investigation of the SMTP traffic between `10.0.0.179` and `185.61.152.63` provided evidence linking the later activity to the earlier executable download. The SMTP stream contained references to `jojo@big3.icu`, followed by the transmission of system and hardware information and a large block of data appearing to be an attachment.

Based on the evidence identified during the investigation, the original hypothesis of potential data exfiltration is **supported**.

The evidence indicates a likely sequence of **payload delivery → information collection → potential data exfiltration**:

1. `10.0.0.179` downloaded `/jojo.exe` from `198.12.66.108`.
2. Later activity occurred between `10.0.0.179` and `185.61.152.63`.
3. The SMTP traffic referenced `jojo@big3.icu`.
4. System and hardware information was transmitted from the internal host.
5. A large block of additional data was subsequently transmitted as an apparent attachment.

The investigation therefore indicates likely malicious activity involving payload delivery, host information collection and subsequent transmission of data to an external SMTP server.

Further analysis of the transferred executable and the apparent attachment would be required to determine exactly what information was collected, what the executable did, and whether the transmitted data can be definitively classified as sensitive information.
