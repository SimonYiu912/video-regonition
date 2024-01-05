<template>
  <v-menu v-model="menu" :close-on-content-click="false" location="start">
    <template v-slot:activator="{ props }">
      <v-btn icon="mdi-robot" v-bind="props" style="position: absolute; bottom: 20px; right: 20px;"></v-btn>
    </template>

    <v-card height="650" width="420" title="eClass AI" subtitle="Your eClass Intelligent Assistant"
      class="d-flex flex-column align-center">

      <v-btn-toggle v-model="text" color="indigo-darken-4" group rounded="xl">
        <v-btn value="Text">
          Text
        </v-btn>

        <v-btn value="Video">
          Video
        </v-btn>

        <v-btn value="Sound">
          Sound
        </v-btn>
      </v-btn-toggle>

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
