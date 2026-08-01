---
title: Sine Wav Generator
author: philip
date: 2025-06-06 13:32:00 +0800
categories: [Tutorial]
tags: [Rust, chunks, sin, audio, file]
render_with_liquid: false
---



Continuing with my exploration of fundementals and chunk file types, I decided to build a sine wav file generator in Rust that takes an input of length in seconds and a frequency to build a WAV file that follows the WAV format and can be played back.

In this first part, I will be explaining how to create the generator using Rust Then in the second part, I will explain how to deploy it online using Web Assembly. 

#### Set up

Let's set up a new project by running `cargo new sine-wav-generator` in terminal.

This will create the `main.rs` entry point. Since we will eventually be building this to be used in Wasm, let's also create a `lib.rs` file in the `src`  folder. This will allow us to run the app using `cargo run` and then also compile it to Wasm later.



#### What is a Sine Wave?







#### What is a Wav file?

A Wav file, or, Waveform Audio File, is a format used for storing uncompressed, raw audio data.

Similarly to PNG files which I've written about **here**, Wav files are composed of structured chunks:

- **Riff** chunk header
  - `ChunkID`  - 4 bytes: the letters `RIFF` in ASCII form.
  - `ChunkSize` - 4 bytes: the size of the file in bytes minus the 8 bytes for this field and the `ChunkID`
  - `Format` - 4 bytes: the letters `WAVE`
- **fmt**
  - `Subchunk1ID` - 4 bytes: the letters `fmt `
  - `Subchunk1Size`- 4 bytes: the value `16` if using PCM. This designates the size of the fmt chunk as being 16 bytes (not including this and the `Subchunk1ID`).
  - `AudioFormat` - 2 bytes:  `1` for PCM. Values other than 1 indicate some form of compression.
  - `NumChannels` - 2 bytes:  `1` indicating Mono, `2`  Stereo, etc.
  - `SampleRate` - 4 bytes:  the sample rate, typically 44100 or 48000
  - `ByteRate` - 4 bytes 

-  **data**
  - `Subchunk2ID` - 4 bytes: the letters `data`
  - `Subchunk2Size`  - 4 bytes: The number of bytes in the data. Calculated by `NumSamples * NumChannels * (bits_per_sample / 8)` 
  - `Data` - The sound data. 



These three chunks make up the structure of a Wav file
