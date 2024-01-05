<template>
  <div class="webcam">
    <div class="action-buttons">
      <!-- <v-btn v-if="!isRecording && !isLoading" @click="handleRecording" :disabled="enableAutoDetection">Start
        Recording</v-btn> -->
      <v-switch v-model="enableAutoDetection" label="Auto Detection" color="primary" inset hide-details></v-switch>
    </div>
    <div v-if="isRecording">
      <p>Recording...</p>
    </div>
    <video ref="videoElement" autoplay playsinline style="width: 380px;"></video>
    <canvas ref="canvasElement" style="display:none;"></canvas>

    <!-- <div v-if="capturedImages.length">
      <h2>Captured Images</h2>
      <div class="images-container">
        <img v-for="(src, index) in capturedImages" :key="index" :src="src" />
      </div>
    </div> -->

    <v-list v-if="conversations.length" class="conversations" lines="two">
      <v-divider></v-divider>
      <template v-for="conversation in conversations" :key="conversation.id">
        <v-list-item
          :prepend-avatar="conversation.role === 'user' ? 'https://cdn.vuetifyjs.com/images/john.jpg' : 'https://cdn.iconscout.com/icon/premium/png-256-thumb/ai-robot-5-1089411.png'">
          {{ conversation.content }}
        </v-list-item>
        <v-divider></v-divider>
      </template>
    </v-list>

  </div>
</template>
<script>
import axios from 'axios';

export default {
  name: 'WebcamStream',
  data() {
    return {
      stream: null,
      audioContext: null,
      audioProcessor: null,
      threshold: 0.1,
      recorder: null,
      recordedChunks: [],
      recordedBlob: null,
      capturedImages: [],
      conversations: [],
      isRecording: false,
      isLoading: false,
      enableAutoDetection: false
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
      this.conversations.push({
        role: 'user',
        content: 'Describe What is this guy doing'
      })
      let content = [
        {
          "type": "text",
          "text": "Use 20 words to describe What is this guy doing in the images I provide to you? Please pay attention to the sequence of the images."
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
          this.conversations.push({
            role: 'ai',
            content: response.data.choices[0].message.content
          })
        })
        .catch((error) => {
          console.log(error);
        }).finally(() => {
          this.isRecording = false;
          this.isLoading = false;
        })
    },
    async initWebcamStream() {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
        this.$refs.videoElement.srcObject = stream;
        this.stream = stream;
        this.setupAudioProcessing();
      } catch (error) {
        console.error('Error accessing the webcam', error);
      }
    },
    setupAudioProcessing() {
      if (!this.stream) {
        return;
      }

      this.audioContext = new AudioContext();
      const source = this.audioContext.createMediaStreamSource(this.stream);
      this.audioProcessor = this.audioContext.createScriptProcessor(1024, 1, 1);

      source.connect(this.audioProcessor);
      this.audioProcessor.connect(this.audioContext.destination);

      this.audioProcessor.onaudioprocess = (event) => {
        if (!this.enableAutoDetection) {
          return;
        }
        const input = event.inputBuffer.getChannelData(0);
        let sum = 0.0;
        for (let i = 0; i < input.length; ++i) {
          sum += input[i] * input[i];
        }
        let volume = Math.sqrt(sum / input.length);
        if (volume > this.threshold && !this.isRecording) {
          this.handleRecording();
        }
      };
    },
    async handleRecording() {
      this.isRecording = true;
      this.isLoading = true;

      if (this.stream) {
        this.isLoading = false;
        this.recorder = new MediaRecorder(this.stream);

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
          this.captureImages();
        };

        this.recorder.start();

        // Stop recording after 5 seconds
        setTimeout(() => {
          this.recorder.stop();
        }, 8000);
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
      a.download = 'recorded-video.mp4';
      a.click();
      window.URL.revokeObjectURL(url);
    },
    captureImages() {
      const video = document.createElement('video');
      video.src = URL.createObjectURL(this.recordedBlob);
      video.load();
      video.play();

      // Wait for the video to be ready, then capture the images
      video.oncanplaythrough = () => {
        const canvas = this.$refs.canvasElement;
        canvas.width = video.videoWidth;
        canvas.height = video.videoHeight;
        const context = canvas.getContext('2d');

        // Calculate the time interval for capturing images
        const captureInterval = 5 / 5; // Duration is 5 seconds, so we divide by 5 to get 5 images

        // Clear previously captured images
        this.capturedImages = [];

        for (let i = 0; i < 8; i++) {
          setTimeout(() => {
            if (i > 2) {
              context.drawImage(video, 0, 0, canvas.width, canvas.height);
              this.capturedImages.push(canvas.toDataURL('image/png'));
              if (i === 7) {
                video.pause();
                this.chatCompletion();
              }
            }
          }, i * captureInterval * 1000); // Multiply by 1000 to convert seconds to milliseconds
        }
      };
    }
  },
  async mounted() {
    await this.initWebcamStream();
  },
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

.conversations {
  margin: 10px;
  width: 100%;
}

.v-divider {
  margin: 0 !important;
  border: 1px solid #000 !important;
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