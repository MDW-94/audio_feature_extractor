# audio_feature_extractor

A simple audio analysis app for audio files (MP3, WAV, etc).

User uploads a file to output JSON object of extracted audio features: BPM, spectral features and more.

JSON can be downloaded as a file or copied directly from logged display.

## Setup & Installation

```
git clone https://github.com/MDW-94/audio_feature_extractor.git
```

## MVP

- take an audio file
- load it with Librosa
- extract the tempo and some spectral features
- print them out or save them to a JSON file

### MVP+

• Tempo — the BPM of a piece of audio
• Spectral features — what frequencies are dominant, essentially the “colour” of the sound
• Onset detection — where beats or events occur in time
• Mel-frequency cepstral coefficients (MFCCs) — a compact representation of the sound that’s widely used in audio ML and speech recognition

## Resources

- [Librosa](https://github.com/librosa/librosa)
- ...
