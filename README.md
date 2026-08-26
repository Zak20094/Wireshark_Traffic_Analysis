# Wireshark PCAP Traffic Analysis

## About

This project involved analysing a provided PCAP file to investigate network traffic and identify any activity that appeared suspicious or unusual.

The investigation was mainly carried out using TShark from the Linux terminal, with Wireshark used for further investigation of interesting traffic.

## Objective

The main objectives were to:

* Identify the main internal host in the capture.
* Find the external IP addresses it was communicating with.
* Investigate the traffic associated with those connections.
* Look at DNS, HTTP, TLS and SMTP traffic.
* Follow interesting traffic further using Wireshark.
* Document the investigation and evidence found.

## Tools Used

* TShark
* Wireshark
* Linux

## Project Structure

```text
Wireshark_Traffic_Analysis/
├── findings/
│   └── analysis.md
├── screenshots/
└── README.md
```

`analysis.md` contains the investigation, reasoning and conclusion.

The `screenshots` folder contains screenshots taken during the investigation.

## PCAP

The investigation was performed on a provided traffic capture from a malware traffic analysis exercise.
