<template>
  <div class="webcam">
    <div v-if="isLoading" class="loading-screen">
      <p>Loading...</p>
    </div>
    <div class="action-buttons">
      <button @click="handleRecording">Start Recording</button>
      <!-- <button v-if="recordedBlob" @click="downloadVideo">Download Video</button> -->
      <!-- <button v-if="recordedBlob" @click="captureImages">Capture Images</button> -->
      <!-- <button v-if="recordedBlob" @click="speechToText">Speech to text</button> -->
    </div>
    <video ref="videoElement" autoplay playsinline></video>
    <canvas ref="canvasElement" style="display:none;"></canvas>

    <div v-if="capturedImages.length">
      <h2>Captured Images</h2>
      <div class="images-container">
        <img v-for="(src, index) in capturedImages" :key="index" :src="src" />
      </div>
    </div>
    <!-- <button v-if="capturedImages.length" @click="chatCompletion">Chat completion</button> -->
    <div v-if="output">
      {{ output }}
    </div>
  </div>
</template>
<script>
import axios from 'axios';

export default {
  name: 'WebcamStream',
  data() {
    return {
      recorder: null,
      recordedChunks: [],
      recordedBlob: null,
      capturedImages: [],
      output: null,
      isLoading: false,
    };
  },
  methods: {
    async speechToText() {
      const file = new File([this.recordedBlob], 'audio.wav', { type: 'audio/wav' })
      const formData = new FormData();
      formData.append('file', file);
      formData.append('model', 'whisper-1');

      try {
        const response = await axios.post('https://api.openai.com/v1/audio/transcriptions', formData, {
          headers: {
            'Authorization': `Bearer ${process.env.VUE_APP_OPENAI_SECRET_KEY}`,
            'Content-Type': 'multipart/form-data'
          }
        });
        console.log(response.data);
        // Handle success
      } catch (error) {
        console.error(error);
        // Handle error
      }
    },
    chatCompletion() {
      this.isLoading = true;
      let content = [
        {
          "type": "text",
          "text": "Use 50 words to tell me what¡¦s this guy doing using the images I provide to you? Please pay attention to the sequence of the images."
        }
      ]
      for (let i = 0; i < this.capturedImages.length; i++) {
        content.push({
          "type": "image_url",
          "image_url": {
            "url": this.capturedImages[i]
          }
        })
      }

      let data = JSON.stringify({
        "model": "gpt-4-vision-preview",
        "messages": [
          {
            "role": "user",
            "content": content
          }
        ]
      });

      let config = {
        method: 'post',
        maxBodyLength: Infinity,
        url: 'https://api.openai.com/v1/chat/completions',
        headers: {
          'OpenAI-Beta': 'assistants=v1',
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${process.env.VUE_APP_OPENAI_SECRET_KEY}`
        },
        data: data
      };

      axios.request(config)
        .then((response) => {
          console.log(JSON.stringify(response.data));
          this.output = JSON.stringify(response.data.choices[0].message.content);
        })
        .catch((error) => {
          console.log(error);
        }).finally(() => {
          this.isLoading = false;
        })
    },
    async getWebcamStream() {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
        this.$refs.videoElement.srcObject = stream;
        return stream;
      } catch (error) {
        console.error('Error accessing the webcam', error);
      }
    },

    async handleRecording() {
      this.isLoading = true;
      const stream = await this.getWebcamStream();

      if (stream) {
        this.isLoading = false;
        this.recorder = new MediaRecorder(stream);

        this.recorder.ondataavailable = event => {
          if (event.data.size > 0) {
            this.recordedChunks.push(event.data);
          }
        };

        this.recorder.onstop = () => {
          this.recordedBlob = new Blob(this.recordedChunks, {
            type: 'video/mp4'
          });
          this.recordedChunks = [];
          stream.getTracks().forEach(track => track.stop());
          this.captureImages();
        };

        this.recorder.start();

        // Stop recording after 5 seconds
        setTimeout(() => {
          this.recorder.stop();
        }, 5000);
      }
    },

    downloadVideo() {
      if (!this.recordedBlob) {
        return;
      }

      const url = URL.createObjectURL(this.recordedBlob);
      const a = document.createElement('a');
      document.body.appendChild(a);
      a.style = 'display: none';
      a.href = url;
      a.download = 'recorded-video.webm';
      a.click();
      window.URL.revokeObjectURL(url);
    },
    captureImages() {
      // Create a video element
      const video = document.createElement('video');
      video.src = URL.createObjectURL(this.recordedBlob);

      video.load();

      // When the metadata is loaded, set up the canvas size
      video.onloadedmetadata = () => {
        const canvas = this.$refs.canvasElement;
        canvas.width = video.videoWidth;
        canvas.height = video.videoHeight;
        const context = canvas.getContext('2d');

        // Initialize captured images array
        this.capturedImages = [];

        // This function will capture the images at each interval
        const captureImage = (index) => {
          context.drawImage(video, 0, 0, canvas.width, canvas.height);
          this.capturedImages.push(canvas.toDataURL('image/png'));

          if (index >= 4) {
            video.pause();
            if (this.chatCompletion) {
              this.chatCompletion();
            }
          } else {
            setTimeout(() => captureImage(index + 1), 1000);
          }
        };

        // Start capturing images one second apart
        setTimeout(() => captureImage(0), 2000);
      };

      // Play the video
      video.play();
    }
  }
};
</script>

<style scoped>
.webcam {
  font-family: 'Arial', sans-serif;
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.action-buttons {
  display: flex;
  justify-content: center;
  gap: 10px;
  margin: 10px 0;
}

h1 {
  color: #333;
}

video {
  border: 3px solid #666;
  border-radius: 5px;
}

button {
  background-color: #4CAF50;
  color: white;
  padding: 14px 20px;
  margin: 8px 0;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  width: auto;
}

button:hover {
  background-color: #45a049;
}

.images-container {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 10px;
  padding: 10px;
}

.images-container img {
  max-width: 150px;
  max-height: 150px;
  border: 2px solid #ddd;
  border-radius: 4px;
  padding: 5px;
}

.output {
  margin-top: 20px;
  background-color: #f1f1f1;
  padding: 10px;
  border-radius: 4px;
}

.loading-screen {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.7);
  display: flex;
  justify-content: center;
  align-items: center;
  color: white;
  font-size: 2em;
  z-index: 100;
}
</style>