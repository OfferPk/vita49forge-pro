# VITA49Forge Pro User Guide & Developer Manual

## 1\. Quickstart Guide

1. Launch `index.html` in your web browser.  
2. In the "VRT Radio Stream & RF Tuner Config" panel, set your Stream ID (e.g. `0x10000001`).  
3. Configure RF parameters: Center Frequency (`2.4 GHz`), Bandwidth (`40 MHz`), Sample Rate (`50 MSPS`), and RF Gain (`25.0 dB`).  
4. Click **Transmit IF Data Packet** to generate a Type 1 IF Data packet with complex I/Q samples, dual-stage UTC/picosecond timestamps, and a 32-bit Trailer word.  
5. Click **Broadcast Context Packet** to serialize a Type 4 Context packet packaging the RF front-end parameters into 64-bit Radix-20 fixed-point fields.  
6. Toggle "Simulate ADC Over-Range Alert" to test trailer bit 25 alert triggering.  
7. Click **Run Normative Audit** to evaluate all 10 ANSI/VITA 49.0/49.2 protocol compliance checks.

## 2\. Standalone Headless Python CLI

Run automated regression testing and compliance verification:

python3 vita49-cli.py \--audit \--stream 0x10000001 \--freq 2400000000