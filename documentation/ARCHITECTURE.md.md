# ANSI/VITA 49.0 & 49.2 (VITA Radio Transport) Technical Architecture

## 1\. Executive Summary & Protocol Architecture

The VITA Radio Transport (VRT) standard, formalized in ANSI/VITA 49.0 and ANSI/VITA 49.2, defines an open protocol architecture for Software-Defined Radio (SDR) and Radio Frequency (RF) digital distribution. VRT decouples RF digitizers, tuners, and antennas from backend digital signal processing (DSP) algorithms and recorder storage engines.

### Key Architectural Concepts:

1. **32-Bit Word Alignment & Standard Headers**:  
   - Every VRT packet consists of an integer number of 32-bit words.  
   - Header: Packet Type (4b), Class ID included (1b), Trailer included (1b), Timestamp Integer (TSI 2b), Timestamp Fractional (TSF 2b), Packet Count (4b), and Packet Size (16b words).  
2. **Standard Packet Typology**:  
   - **Type 1 (IF Data with Stream ID)**: Conveys digitized real or complex I/Q baseband samples bound to a unique 32-bit Stream ID.  
   - **Type 4 (Context Packet)**: Conveys real-time RF state metadata (Center Frequency, Bandwidth, Reference Level, Gain, Sample Rate, Antenna polarization, and GPS coordinates).  
   - **Type 6 (Control Packet)**: Sends commands to tuners, synthesizers, and attenuators.  
   - **Type 7 (Acknowledge Packet)**: Reports command execution outcomes and RF hardware health.  
3. **Precision Dual-Stage Timestamping**:  
   - Integer Seconds (32-bit UTC / GPS epoch timestamp).  
   - Fractional Seconds (64-bit real-time fractional seconds with picosecond resolution), enabling multi-receiver TDOA (Time Difference of Arrival) geolocation and coherent phased-array beamforming.  
4. **Trailer Word Health & Integrity**:  
   - 32-bit Trailer word provides per-packet status: ADC over-range, sample drop, phase reference lock, and time calibration status.