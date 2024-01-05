<template>
  <v-menu v-model="menu" :close-on-content-click="false" location="start">
    <template v-slot:activator="{ props }">
      <v-btn v-bind="props"
        style="position: absolute; bottom: 20px; right: 20px;width:25px; height: 60px; border-radius: 50%;">
        <img src="https://cdn.iconscout.com/icon/premium/png-256-thumb/ai-robot-5-1089411.png" alt="AI Robot"
          style="width:25px; height: 25px;">
      </v-btn>
    </template>

    <v-card height="650" width="420" class="d-flex flex-column align-center">
      <v-card-title style="width: 100%;">
        <img src="./assets/eClassAI.png" width="100px" />
      </v-card-title>

      <v-card-subtitle style="width: 100%;">
        <strong>Your eClass Intelligent Assistant</strong>
      </v-card-subtitle>

      <v-card-item style="width: 100%;" class="d-flex justify-center">
        <v-btn-toggle v-model="text" color="indigo-darken-4" group rounded="xl">
          <v-btn value=" Text">
            Text
          </v-btn>

          <v-btn value="Video">
            Video
          </v-btn>

          <v-btn value="Sound">
            Sound
          </v-btn>
        </v-btn-toggle>
      </v-card-item>

      <v-card-actions>
        <h2 v-show="text !== 'Video'">Coming soon</h2>
        <WebcamStream v-show="text === 'Video'" />
      </v-card-actions>
    </v-card>
  </v-menu>
</template>

<script>
import WebcamStream from './components/WebcamStream.vue'

export default {
  name: 'App',
  components: {
    WebcamStream
  },
  data: () => ({
    text: 'Video',
    menu: false,
  }),
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
