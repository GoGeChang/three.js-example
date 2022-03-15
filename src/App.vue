<template>
  <div id="app">
    <cannon></cannon>
  </div>
</template>

<script>
// import threeContent from './components/three.vue'
import cannon from "./components/cannon.vue"
import module from "./components/module.vue"

export default {
  components: {  cannon },
  name: 'App',
  mounted () {
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  overflow: hidden;
}
body {
  margin: 0;
  padding: 0;
}
</style>
