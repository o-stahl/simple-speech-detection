<template>
    <div>
      <h1>Simple Speech Detection</h1>
      <div id="status">
        <p>Stream Status: <span>{{ streamStatus }}</span></p>
        <p>Speech Detection: <span>{{ speechStatus }}</span></p>
        <p>Active Modules: RMS, ZCR, Spectral Centroid, Spectral Rolloff, MFCC</p>
      </div>
      <div id="waveform"></div>
      <h2>Recorded Clips</h2>
      <ul id="recordings-list">
        <li v-for="(clip, index) in recordings" :key="index">
          <audio :src="clip.url" controls></audio>
          <a :href="clip.url" :download="'speech-' + index + '.mp4'">Download</a>
          <canvas :id="'chart-' + index" width="400" height="150"></canvas>
          <table>
            <thead>
              <tr>
                <th>Time</th>
                <th>RMS</th>
                <th>ZCR</th>
                <th>Spectral Centroid</th>
                <th>Spectral Rolloff</th>
                <th>MFCC</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(feature, idx) in clip.features" :key="idx">
                <td>{{ feature.time }}</td>
                <td>{{ feature.rms.toFixed(4) }}</td>
                <td>{{ feature.zcr.toFixed(4) }}</td>
                <td>{{ feature.spectralCentroid.toFixed(4) }}</td>
                <td>{{ feature.spectralRolloff.toFixed(4) }}</td>
                <td>{{ feature.mfcc[0].toFixed(4) }}</td>
              </tr>
            </tbody>
          </table>
        </li>
      </ul>
    </div>
  </template>
  
  <script>
  import { ref, onMounted, onUnmounted, nextTick } from 'vue';
  import WaveSurfer from 'wavesurfer.js';
  import Meyda from 'meyda';
  import Chart from 'chart.js/auto';
  
  class KalmanFilter {
    constructor({ R = 0.01, Q = 0.1, A = 1.0, B = 0.0, C = 1.0 } = {}) {
      this.R = R; // noise power desirable
      this.Q = Q; // noise power estimated
      this.A = A;
      this.B = B;
      this.C = C;
      this.cov = NaN;
      this.x = NaN; // estimated signal without noise
    }
  
    filter(z, u = 0) {
      if (isNaN(this.x)) {
        this.x = (1 / this.C) * z;
        this.cov = (1 / this.C) * this.Q * (1 / this.C);
      } else {
        const predX = this.A * this.x + this.B * u;
        const predCov = this.A * this.cov * this.A + this.R;
        const K = predCov * this.C * (1 / (this.C * predCov * this.C + this.Q));
        this.x = predX + K * (z - this.C * predX);
        this.cov = predCov - K * this.C * predCov;
      }
      return this.x;
    }
  }
  
  export default {
    setup() {
      const streamStatus = ref('Initializing...');
      const speechStatus = ref('Listening...');
      const audioContext = ref(null);
      const mediaRecorder = ref(null);
      const chunks = ref([]);
      const recordings = ref([]);
      const recording = ref(false);
      const silenceCooldown = ref(0);
      const waveSurfer = ref(null);
      const featureData = ref([]);
      const preSpeechBuffer = ref([]);
      const charts = ref({});
  
      const silenceTimeoutMs = 500;
      const sampleRate = 44100;
      const bufferLength = 4096;
      const silenceCooldownThreshold = Math.ceil((silenceTimeoutMs / 1000) * (sampleRate / bufferLength));
  
      const preSpeechBufferSize = 10;
  
      const backgroundLevels = ref({
        rms: [],
        zcr: [],
        spectralCentroid: [],
        spectralRolloff: [],
        mfcc: []
      });
  
      const kalmanFilters = {
        rms: new KalmanFilter(),
        zcr: new KalmanFilter(),
        spectralCentroid: new KalmanFilter(),
        spectralRolloff: new KalmanFilter(),
        mfcc: new KalmanFilter()
      };
  
      onMounted(() => {
        document.body.addEventListener('click', initAudio, { once: true });
      });
  
      onUnmounted(() => {
        if (audioContext.value) audioContext.value.close();
        if (mediaRecorder.value && mediaRecorder.value.state !== 'inactive') mediaRecorder.value.stop();
      });
  
      async function initAudio() {
        try {
          audioContext.value = new (window.AudioContext)();
          const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
          streamStatus.value = 'Stream started';
  
          const source = audioContext.value.createMediaStreamSource(stream);
          const processor = audioContext.value.createScriptProcessor(bufferLength, 1, 1);
  
          processor.onaudioprocess = (event) => {
            const inputData = event.inputBuffer.getChannelData(0);
            analyzeAudio(inputData);
            simulateRealTimeVisualization(inputData);
          };
  
          source.connect(processor);
          processor.connect(audioContext.value.destination);
  
          mediaRecorder.value = new MediaRecorder(stream);
          mediaRecorder.value.ondataavailable = (event) => {
            chunks.value.push(event.data);
          };
          mediaRecorder.value.onstop = saveRecording;
  
          waveSurfer.value = WaveSurfer.create({
            container: '#waveform',
            waveColor: 'violet',
            progressColor: 'purple',
            interact: false,
            cursorWidth: 0,
          });
        } catch (error) {
          streamStatus.value = `Error: ${error.message}`;
          console.error('Error accessing microphone:', error);
        }
      }
  
      function analyzeAudio(data) {
        const features = Meyda.extract(['rms', 'zcr', 'spectralCentroid', 'spectralRolloff', 'mfcc'], data);
        if (!features || typeof features.rms === 'undefined') return;
  
        // Apply Kalman filter to each feature
        const filteredFeatures = {
          time: new Date().toLocaleTimeString(),
          rms: kalmanFilters.rms.filter(features.rms),
          zcr: kalmanFilters.zcr.filter(features.zcr),
          spectralCentroid: kalmanFilters.spectralCentroid.filter(features.spectralCentroid),
          spectralRolloff: kalmanFilters.spectralRolloff.filter(features.spectralRolloff),
          mfcc: features.mfcc.map((mfcc, index) => kalmanFilters.mfcc.filter(mfcc))
        };
  
        featureData.value.push(filteredFeatures);
  
        preSpeechBuffer.value.push(filteredFeatures);
        if (preSpeechBuffer.value.length > preSpeechBufferSize) {
          preSpeechBuffer.value.shift();
        }
  
        updateBackgroundNoise(filteredFeatures);
  
        const speechDetected = detectSpeech(filteredFeatures);
        if (speechDetected) {
          if (!recording.value) startRecording();
          silenceCooldown.value = silenceCooldownThreshold;
          speechStatus.value = 'Speech detected';
        } else {
          if (recording.value) {
            silenceCooldown.value--;
            if (silenceCooldown.value <= 0) stopRecording();
            speechStatus.value = 'No speech detected';
          }
        }
      }
  
      function updateBackgroundNoise(features) {
        if (backgroundLevels.value.rms.length < 100) { // Use first 100 samples for background noise estimation
          backgroundLevels.value.rms.push(features.rms);
          backgroundLevels.value.zcr.push(features.zcr);
          backgroundLevels.value.spectralCentroid.push(features.spectralCentroid);
          backgroundLevels.value.spectralRolloff.push(features.spectralRolloff);
          backgroundLevels.value.mfcc.push(features.mfcc[0]);
        } else {
          backgroundLevels.value.rms.shift();
          backgroundLevels.value.zcr.shift();
          backgroundLevels.value.spectralCentroid.shift();
          backgroundLevels.value.spectralRolloff.shift();
          backgroundLevels.value.mfcc.shift();
  
          backgroundLevels.value.rms.push(features.rms);
          backgroundLevels.value.zcr.push(features.zcr);
          backgroundLevels.value.spectralCentroid.push(features.spectralCentroid);
          backgroundLevels.value.spectralRolloff.push(features.spectralRolloff);
          backgroundLevels.value.mfcc.push(features.mfcc[0]);
        }
      }
  
      function calculateThresholds() {
        const meanAndStd = (arr) => {
          const mean = arr.reduce((a, b) => a + b, 0) / arr.length;
          const std = Math.sqrt(arr.map(x => Math.pow(x - mean, 2)).reduce((a, b) => a + b) / arr.length);
          return { mean, std };
        };
  
        const rmsStats = meanAndStd(backgroundLevels.value.rms);
        const zcrStats = meanAndStd(backgroundLevels.value.zcr);
        const spectralCentroidStats = meanAndStd(backgroundLevels.value.spectralCentroid);
        const spectralRolloffStats = meanAndStd(backgroundLevels.value.spectralRolloff);
        const mfccStats = meanAndStd(backgroundLevels.value.mfcc);
  
        return {
          rms: rmsStats.mean + 2 * rmsStats.std,
          zcr: zcrStats.mean + 2 * zcrStats.std,
          spectralCentroid: spectralCentroidStats.mean + 2 * spectralCentroidStats.std,
          spectralRolloff: spectralRolloffStats.mean + 2 * spectralRolloffStats.std,
          mfcc: mfccStats.mean + 2 * mfccStats.std
        };
      }
  
      function detectSpeech(features) {
        const thresholds = calculateThresholds();
        return features.mfcc[0] > thresholds.mfcc && 
               (features.rms > thresholds.rms ||
                features.zcr > thresholds.zcr ||
                features.spectralCentroid > thresholds.spectralCentroid ||
                features.spectralRolloff > thresholds.spectralRolloff);
      }
  
      function simulateRealTimeVisualization(data) {
        const buffer = audioContext.value.createBuffer(1, data.length, audioContext.value.sampleRate);
        buffer.copyToChannel(data, 0);
        const blob = new Blob([buffer], { type: 'audio/wav' });
        waveSurfer.value.loadBlob(blob);
      }
  
      function startRecording() {
        chunks.value = [];
        featureData.value = [...preSpeechBuffer.value]; // Include pre-speech buffer
        mediaRecorder.value.start();
        recording.value = true;
      }
  
      function stopRecording() {
        mediaRecorder.value.stop();
        recording.value = false;
      }
  
      function saveRecording() {
        const blob = new Blob(chunks.value, { type: 'audio/mp4' });
        const url = URL.createObjectURL(blob);
        recordings.value.push({ url, features: [...featureData.value] });
        nextTick(renderCharts);
      }
  
      function renderCharts() {
        recordings.value.forEach((recording, index) => {
          const canvasId = `chart-${index}`;
          const ctx = document.getElementById(canvasId).getContext('2d');
  
          if (charts.value[canvasId]) {
            charts.value[canvasId].destroy();
          }
  
          charts.value[canvasId] = new Chart(ctx, {
            type: 'line',
            data: {
              labels: recording.features.map((_, i) => i),
              datasets: [
                {
                  label: 'RMS',
                  data: recording.features.map(f => f.rms),
                  borderColor: 'rgba(75, 192, 192, 1)',
                  borderWidth: 1,
                  fill: false,
                },
                {
                  label: 'ZCR',
                  data: recording.features.map(f => f.zcr),
                  borderColor: 'rgba(153, 102, 255, 1)',
                  borderWidth: 1,
                  fill: false,
                },
                {
                  label: 'Spectral Centroid',
                  data: recording.features.map(f => f.spectralCentroid),
                  borderColor: 'rgba(255, 159, 64, 1)',
                  borderWidth: 1,
                  fill: false,
                },
                {
                  label: 'Spectral Rolloff',
                  data: recording.features.map(f => f.spectralRolloff),
                  borderColor: 'rgba(255, 205, 86, 1)',
                  borderWidth: 1,
                  fill: false,
                },
                {
                  label: 'MFCC',
                  data: recording.features.map(f => f.mfcc[0]),
                  borderColor: 'rgba(54, 162, 235, 1)',
                  borderWidth: 1,
                  fill: false,
                },
              ]
            },
            options: {
              responsive: true,
              scales: {
                x: {
                  type: 'linear',
                  position: 'bottom'
                }
              }
            }
          });
        });
      }
  
      return {
        streamStatus,
        speechStatus,
        recordings,
      };
    }
  };
  </script>
  
  <style scoped>
  #status {
    padding: 10px;
    background-color: #fff;
    margin: 20px;
    border-radius: 8px;
    box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
  }
  
  #status p {
    margin: 5px 0;
  }
  
  #waveform {
    width: 100%;
    height: 200px;
    background-color: #333;
  }
  
  audio {
    width: 100%;
    outline: none;
  }
  
  table {
    width: 100%;
    border-collapse: collapse;
    margin-top: 10px;
  }
  
  table, th, td {
    border: 1px solid #ddd;
  }
  
  th, td {
    padding: 8px;
    text-align: left;
  }
  
  th {
    background-color: #f4f4f4;
  }
  </style>
  