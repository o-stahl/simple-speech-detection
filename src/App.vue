<template>
  <div id="app">
    <SpeechDetection />
  </div>
</template>

<script>
import SpeechDetection from './components/SpeechDetection.vue';

export default {
  name: 'App',
  components: {
    SpeechDetection
  }
};
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
