# AFSK Messenger (Web)

## LEGAL DISCLAIMER

This software is a tool for generating and decoding audio-frequency tones. It does not itself transmit radio frequency energy. Any use of this tool to transmit over radio frequencies must comply with all applicable laws and regulations in your jurisdiction.

- **Amateur (ham) radio bands** may only be used by individuals holding a valid amateur radio license issued by the relevant regulatory authority (e.g., FCC in the United States, OFCOM in the UK, ACMA in Australia). Unlicensed transmission on amateur bands is illegal.
- **Family Radio Service (FRS), General Mobile Radio Service (GMRS), Personal Mobile Radio (PMR), and similar license-free or license-by-rule services** have their own power, channel, and operational restrictions. You are responsible for understanding and following those rules.
- **Business, public safety, and other licensed radio services** require authorization from the license holder.
- Using this tool for any unlawful purpose, including but not limited to interference with licensed communications, is strictly prohibited.

This software is provided for educational and experimental use. The authors assume no liability for any violations of law resulting from its use.

---

A browser-based text-over-radio application that encodes messages as AFSK (Audio Frequency-Shift Keying) tones and decodes received audio from a microphone. Designed to work with handheld radios such as Baofeng UV-5R by coupling the computer speaker to the radio microphone and the radio speaker to the computer microphone.

## How it works

### Modulation

The system uses Bell 103-compatible AFSK at 50 baud:

- Mark (bit 1): 1200 Hz sine wave for 20 ms
- Space (bit 0): 2200 Hz sine wave for 20 ms

Each symbol is 20 ms long, providing 24 cycles of the mark tone or 44 cycles of the space tone per symbol. This long symbol period makes the modulation highly robust to noise, distortion, and frequency drift.

### Frame format

Each transmission encodes a single frame:

```
[ 16-bit preamble 0xAA55 ]
[  8-bit payload length    ]
[  N-byte payload (UTF-8)  ]
[ 16-bit CRC-CCITT         ]
```

The preamble allows the receiver to locate the start of the frame in the decoded bitstream. The CRC-CCITT provides error detection. If the CRC does not match, the frame is silently discarded.

### Transmit process

1. The user types a message and triggers send.
2. The message is encoded as UTF-8 bytes.
3. A frame is constructed: preamble, length byte, payload, CRC.
4. The frame is converted to bits (MSB first) and each bit is modulated as a sine tone at 1200 Hz (bit 1) or 2200 Hz (bit 0).
5. The audio is assembled as: lead tone (1000 Hz, 1.0 s) + silence gap (0.3 s) + data tones + tail tone (1000 Hz, 0.5 s).
6. The combined audio is played through the speaker. The user must key the radio PTT during the lead tone and release it after the tail tone ends.

### Receive process

1. The browser captures microphone audio via the Web Audio API (ScriptProcessorNode callback).
2. For each 20 ms symbol window, the energy at 1200 Hz and 2200 Hz is computed by correlating the samples against pre-computed reference sinusoids.
3. If the 1200 Hz energy exceeds the 2200 Hz energy, the bit is decoded as 1; otherwise 0.
4. Bits are accumulated in a stream. Every new bit is checked against the known 16-bit preamble pattern.
5. When the preamble is matched, the receiver locks onto the frame and reads the length byte, payload, and CRC.
6. If the CRC is valid, the payload is decoded as UTF-8 text and displayed.
7. The receiver returns to preamble search after the frame completes (success or CRC failure).

### Timing

At 50 baud with 255-byte maximum payload:

- Lead tone: 1.0 s
- Gap: 0.3 s
- Data for maximum payload: (16 + 8 + 2040 + 16) / 50 = 41.6 s
- Tail tone: 0.5 s
- Total maximum transmission time: ~43.4 s

## File structure

- `index.html` — single self-contained HTML file. No build step, no dependencies. Open in any modern browser.

## Browser compatibility

Requires a browser with:

- Web Audio API (AudioContext, ScriptProcessorNode)
- getUserMedia (microphone access)
- TextEncoder / TextDecoder
- ES6 (arrow functions, let/const not used; var-based for maximum compatibility)

Tested on desktop Chrome, Firefox, Edge, and mobile Chrome for Android. Safari iOS may require a user gesture to create the AudioContext; the "Start Mic" button handles this.

## Usage

1. Open `index.html` on both devices (or deploy to GitHub Pages).
2. Tap **Start Mic** to grant microphone access and begin listening.
3. Type a message and press Enter (desktop) or tap **Send** (mobile).
4. Immediately key the PTT on the radio. Hold until the progress bar fills and the status says "Ready".
5. The receiving device will show the decoded message when the frame is successfully received.

### Controls

- **Start Mic / Stop Mic** — toggles microphone capture. Clears receiver state on stop.
- **Send** — encodes and transmits the message.
- **Enter key** (desktop, without Shift) — sends the message.
- **Shift+Enter** — inserts a newline without sending.

### Indicators

- **Sync dot**: Red (waiting for signal), Yellow (receiving bits, preamble found), Green (message decoded successfully), Blue (transmitting).
- **Volume meter**: Green (good level), Yellow/Orange (moderate), Red (clipping or too loud). The bar height corresponds to the RMS amplitude of the incoming audio.
- **Progress bar**: Appears during transmission and fills from left to right.

## Developer notes

### CRC-CCITT

The frame uses CRC-CCITT (polynomial 0x1021) with initial value 0xFFFF, computed over the payload bytes only. The CRC is appended as two bytes, high byte first.

### Preamble detection

The preamble 0xAA55 = 0b1010101001010101 is transmitted as 16 bits (MSB first) at the start of the data section. The sliding-window detector compares the last 16 decoded bits against this pattern. Because of the long symbol period and the alternating pattern, the preamble is highly unlikely to appear in random noise or in the lead tone (which produces near-constant mark bits due to the 1000 Hz tone being closer to 1200 Hz than 2200 Hz).

### Symbol energy detection

For each symbol window of N samples (N = floor(sampleRate / 50)), the energy at each tone frequency is computed as:

```
E_mark = (sum(samples * cos(2pi * 1200 * t)))^2 + (sum(samples * sin(2pi * 1200 * t)))^2
E_space = (sum(samples * cos(2pi * 2200 * t)))^2 + (sum(samples * sin(2pi * 2200 * t)))^2
```

No normalization is needed because only the relative energies matter. The reference sinusoids are pre-computed once when the microphone starts and cached for the lifetime of the session.

### Sample rate independence

The reference sinusoids are computed using the actual AudioContext sample rate. This means the implementation works correctly regardless of whether the browser provides 44100 Hz, 48000 Hz, or any other rate. The symbol period in samples is computed as `Math.round(sampleRate / 50)`.

### Limitations

- Maximum payload is 255 bytes. UTF-8 characters beyond byte 255 are silently truncated.
- The implementation is half-duplex: transmission blocks reception (the microphone is still captured but the samples are discarded).
- ScriptProcessorNode is deprecated in the Web Audio spec but remains widely supported. Future migration to AudioWorklet would improve performance and eliminate the deprecation warning.

## License

Original work. No restrictions.
