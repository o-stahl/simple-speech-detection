# simple-speech-detection
*The purpose of this project is to test and flesh out a feature (speech detection) for another project.*
A Vue.js application for real-time simple speech detection (not transcription) and analysis using Web Audio API, AudioWorklet, Meyda, and Chart.js for visualizing some of the data. This project demonstrates real-time audio feature extraction and visualization with capabilities for recording and analyzing speech segments.


# Simple Speech Detection

A Vue.js application for simple speech detection and analysis using Web Audio API, AudioWorklet, Meyda, and Chart.js. This project demonstrates real-time audio feature extraction and visualization with capabilities for recording and analyzing speech segments.

## Features

- Real-time audio streaming and visualization using WaveSurfer.js (not working)
- Audio feature extraction using Meyda
- Speech detection with customizable silence threshold and cooldown
- Recording and playback of detected speech segments
- Visualization of audio features (RMS, ZCR, Spectral Centroid, Spectral Rolloff, MFCC) using Chart.js

## Getting Started

### Prerequisites

- Node.js (>= 14.x)
- npm

### Installation

1. Clone the repository:

   ```sh
   git clone https://github.com/your-username/simple-speech-detection.git
   cd simple-speech-detection

2. Install dependencies:

   ```sh
   npm install

### Running the application

1. Start the development server:

   ```sh
   npm run dev

2. Open your browser and navigate to http://localhost:8080 (or the port specified in your console).

## Usage
Click anywhere on the page to initialize audio.
The application will start listening to the microphone input.
Detected speech segments will be recorded and displayed in a list with playback controls.
Audio features of each segment will be visualized in a chart.

## Customization
Adjust silenceThreshold and silenceTimeoutMs in SpeechDetection.vue to change speech detection sensitivity.
Modify the preSpeechBufferSize to include more or less audio before detected speech segments.

## Contributing
Although contributions are welcome, do note that this is a throwaway project and as such won't likely be maintained to any reasonable degree.

## Acknowledgements
- Vue.js
- WaveSurfer.js
- Meyda
- Chart.js
 
