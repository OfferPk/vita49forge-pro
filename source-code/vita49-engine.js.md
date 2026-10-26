/\*\*  
 \* ANSI/VITA 49.0 & 49.2 VITA Radio Transport (VRT) Core Protocol Engine  
 \* Implements 32-bit word packet framing for IF Data, Context, and Control packets,  
 \* picosecond fractional timestamping, I/Q digital sample packing, and Trailer words.  
 \*/

// VITA 49 Packet Types (Header Bits 31:28)  
const VITA49\_PACKET\_TYPES \= {  
  IF\_DATA: 0x0,             // IF Data without Stream ID  
  IF\_DATA\_WITH\_STREAM: 0x1, // IF Data with Stream ID  
  EXT\_DATA: 0x2,            // Extension Data without Stream ID  
  EXT\_DATA\_WITH\_STREAM: 0x3,// Extension Data with Stream ID  
  CONTEXT: 0x4,             // Context Packet  
  EXT\_CONTEXT: 0x5,         // Extension Context Packet  
  CONTROL: 0x6,             // Control Packet (VITA 49.2)  
  ACKNOWLEDGE: 0x7          // Acknowledge Packet (VITA 49.2)  
};

// Timestamp Types  
const TIMESTAMP\_TYPES \= {  
  NONE: 0x0,  
  UTC: 0x1,  
  GPS: 0x2,  
  POSIX: 0x3  
};

class VITA49Engine {  
  constructor(streamId \= 0x10000001) {  
    this.streamId \= streamId;  
    this.packetCount \= 0; // 4-bit modulo-16 counter  
  }

  getNextPacketCount() {  
    const cnt \= this.packetCount;  
    this.packetCount \= (this.packetCount \+ 1\) & 0x0F;  
    return cnt;  
  }

  /\*\*  
   \* Encodes a 32-bit VITA 49 Standard Header  
   \* Bits 31:28 \- Packet Type (4b)  
   \* Bit 27     \- Class ID Included (1b)  
   \* Bit 26     \- Trailer Included (1b)  
   \* Bits 25:24 \- Reserved  
   \* Bits 23:22 \- Integer Seconds Timestamp Type (2b: 00=None, 01=UTC, 10=GPS, 11=POSIX)  
   \* Bits 21:20 \- Fractional Seconds Timestamp Type (2b: 00=None, 01=Sample Count, 10=Real-Time Picoseconds, 11=Free-running)  
   \* Bits 19:16 \- Packet Count (4b modulo-16)  
   \* Bits 15:0  \- Packet Size in 32-bit words (16b)  
   \*/  
  static encodeHeader(pktType, hasClassId, hasTrailer, tsiType, tsfType, pktCount, packetSizeWords) {  
    let hdr \= 0;  
    hdr \|= (pktType & 0x0F) \<\< 28;  
    hdr \|= (hasClassId ? 1 : 0\) \<\< 27;  
    hdr \|= (hasTrailer ? 1 : 0\) \<\< 26;  
    hdr \|= (tsiType & 0x03) \<\< 22;  
    hdr \|= (tsfType & 0x03) \<\< 20;  
    hdr \|= (pktCount & 0x0F) \<\< 16;  
    hdr \|= (packetSizeWords & 0xFFFF);  
    return hdr \>\>\> 0;  
  }

  /\*\*  
   \* Encodes a 32-bit VITA 49 Trailer Word (Section 9\)  
   \* Bit 31 \- Calibrated Time  
   \* Bit 30 \- Valid Data  
   \* Bit 29 \- Reference Lock  
   \* Bit 28 \- AGC / MGC  
   \* Bit 27 \- Detected Signal  
   \* Bit 26 \- Spectral Inversion  
   \* Bit 25 \- Over-range indicator  
   \* Bit 24 \- Sample Loss indicator  
   \*/  
  static encodeTrailer(isValid \= true, isOverrange \= false, isSampleLoss \= false) {  
    let trailer \= 0;  
    trailer \|= 1 \<\< 31; // Calibrated Time  
    trailer \|= (isValid ? 1 : 0\) \<\< 30; // Valid Data  
    trailer \|= 1 \<\< 29; // Ref Lock  
    if (isOverrange) trailer \|= 1 \<\< 25; // Over-range  
    if (isSampleLoss) trailer \|= 1 \<\< 24; // Sample Loss  
    return trailer \>\>\> 0;  
  }

  /\*\*  
   \* Serializes an IF Data Packet with Stream ID and I/Q samples  
   \*/  
  serializeIfDataPacket(sampleCount \= 64, isOverrange \= false) {  
    const pktCount \= this.getNextPacketCount();  
    const hasStreamId \= true;  
    const hasClassId \= false;  
    const hasTrailer \= true;  
    const tsi \= TIMESTAMP\_TYPES.UTC;  
    const tsf \= 0x02; // Picoseconds

    // Timestamp: UTC seconds (32b) \+ Picoseconds (64b \= 2 words)  
    const now \= Math.floor(Date.now() / 1000);  
    const fracPicosecondsHigh \= 0x00010000;  
    const fracPicosecondsLow \= 0x55AA1234;

    // Generate I/Q complex 16-bit signed samples (1 word \= 1 I/Q pair: 16b I \+ 16b Q)  
    const iqWords \= \[\];  
    for (let i \= 0; i \< sampleCount; i++) {  
      const angle \= (i / sampleCount) \* 2 \* Math.PI;  
      const iVal \= Math.round(Math.cos(angle) \* 32000\) & 0xFFFF;  
      const qVal \= Math.round(Math.sin(angle) \* 32000\) & 0xFFFF;  
      iqWords.push(((iVal \<\< 16\) \| qVal) \>\>\> 0);  
    }

    const trailer \= VITA49Engine.encodeTrailer(true, isOverrange, false);

    // Total words \= Header (1) \+ StreamID (1) \+ TSI (1) \+ TSF (2) \+ Samples (N) \+ Trailer (1)  
    const totalWords \= 1 \+ 1 \+ 1 \+ 2 \+ iqWords.length \+ 1;  
    const header \= VITA49Engine.encodeHeader(  
      VITA49\_PACKET\_TYPES.IF\_DATA\_WITH\_STREAM,  
      hasClassId,  
      hasTrailer,  
      tsi,  
      tsf,  
      pktCount,  
      totalWords  
    );

    const packetWords \= \[  
      header,  
      this.streamId,  
      now,  
      fracPicosecondsHigh,  
      fracPicosecondsLow,  
      ...iqWords,  
      trailer  
    \];

    return {  
      type: 'IF\_DATA\_WITH\_STREAM',  
      packetType: VITA49\_PACKET\_TYPES.IF\_DATA\_WITH\_STREAM,  
      streamId: '0x' \+ (this.streamId \>\>\> 0).toString(16).toUpperCase().padStart(8, '0'),  
      packetCount: pktCount,  
      totalWords: totalWords,  
      totalBytes: totalWords \* 4,  
      samplePairs: sampleCount,  
      headerHex: '0x' \+ (header \>\>\> 0).toString(16).toUpperCase().padStart(8, '0'),  
      trailerHex: '0x' \+ (trailer \>\>\> 0).toString(16).toUpperCase().padStart(8, '0'),  
      words: packetWords  
    };  
  }

  /\*\*  
   \* Serializes a Context Packet broadcasting RF front-end parameters  
   \*/  
  serializeContextPacket(centerFreqHz \= 2400000000, bandwidthHz \= 40000000, sampleRateHz \= 50000000, gainDb \= 25.0) {  
    const pktCount \= this.getNextPacketCount();  
    const hasClassId \= false;  
    const hasTrailer \= false;  
    const tsi \= TIMESTAMP\_TYPES.UTC;  
    const tsf \= 0x02;  
    const now \= Math.floor(Date.now() / 1000);

    // Context Indicator Word (Section 7.1)  
    // Bit 29: Bandwidth (1)  
    // Bit 28: Center Frequency (1)  
    // Bit 27: RF Reference Level / Gain (1)  
    // Bit 26: Sample Rate (1)  
    const contextIndicator \= (1 \<\< 29\) \| (1 \<\< 28\) \| (1 \<\< 27\) \| (1 \<\< 26);

    // Bandwidth: 64-bit Radix-20 fixed point (High 32b, Low 32b)  
    const bwHigh \= Math.floor(bandwidthHz / 4294967296);  
    const bwLow \= bandwidthHz \>\>\> 0;

    // Center Frequency: 64-bit Radix-20 fixed point  
    const cfHigh \= Math.floor(centerFreqHz / 4294967296);  
    const cfLow \= centerFreqHz \>\>\> 0;

    // Gain: 16-bit signed in 0.5 dB steps  
    const gainWord \= (Math.round(gainDb \* 2\) & 0xFFFF) \<\< 16;

    // Sample Rate: 64-bit Radix-20 fixed point  
    const srHigh \= Math.floor(sampleRateHz / 4294967296);  
    const srLow \= sampleRateHz \>\>\> 0;

    // Words: Header (1) \+ StreamID (1) \+ TSI (1) \+ TSF (2) \+ Indicator (1) \+ BW (2) \+ CF (2) \+ Gain (1) \+ SR (2)  
    const totalWords \= 1 \+ 1 \+ 1 \+ 2 \+ 1 \+ 2 \+ 2 \+ 1 \+ 2;  
    const header \= VITA49Engine.encodeHeader(  
      VITA49\_PACKET\_TYPES.CONTEXT,  
      hasClassId,  
      hasTrailer,  
      tsi,  
      tsf,  
      pktCount,  
      totalWords  
    );

    const packetWords \= \[  
      header,  
      this.streamId,  
      now,  
      0x00010000,  
      0x55AA1234,  
      contextIndicator,  
      bwHigh,  
      bwLow,  
      cfHigh,  
      cfLow,  
      gainWord,  
      srHigh,  
      srLow  
    \];

    return {  
      type: 'CONTEXT\_PACKET',  
      packetType: VITA49\_PACKET\_TYPES.CONTEXT,  
      streamId: '0x' \+ (this.streamId \>\>\> 0).toString(16).toUpperCase().padStart(8, '0'),  
      packetCount: pktCount,  
      totalWords: totalWords,  
      totalBytes: totalWords \* 4,  
      centerFreqGhz: (centerFreqHz / 1e9).toFixed(3),  
      bandwidthMhz: (bandwidthHz / 1e6).toFixed(1),  
      sampleRateMsps: (sampleRateHz / 1e6).toFixed(1),  
      gainDb: gainDb,  
      headerHex: '0x' \+ (header \>\>\> 0).toString(16).toUpperCase().padStart(8, '0'),  
      words: packetWords  
    };  
  }  
}

if (typeof module \!== 'undefined' && module.exports) {  
  module.exports \= {  
    VITA49Engine,  
    VITA49\_PACKET\_TYPES,  
    TIMESTAMP\_TYPES  
  };  
}  
