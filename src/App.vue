<template>
  <div id="app">
    <div style="width:600px;height:1000px">
      <MultiLayer/>
    </div>
  </div>
</template>

<script>
import MultiLayer from './components/MultiLayer'

export default {
  name: 'App',
  components: {
    MultiLayer
  },
  
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
