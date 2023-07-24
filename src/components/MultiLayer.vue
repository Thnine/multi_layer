<template>
  <div class="multi_layer_container">
    <svg class="muiti_layer_canva"></svg>
  </div>
</template>

<script>
export default {
  name: 'MultiLayer',
  data () {
    return {
      raw_node:[],//[{'index':'1',...},{'index':'2',...},...]
      raw_link:[],//[{'index':'1',...},{'index':'2',...}]
    }
  },
  methods:{
    setData(nodes,links){//绑定数据
      this.raw_node = JSON.parse(JSON.stringify(nodes));
      this.raw_link = JSON.parse(JSON.stringify(links));
    }
  }
}
</script>

<style scoped>
  .multi_layer_container{
    height:100%;
    width: 100%;
    overflow: auto;
  }
  .muiti_layer_canva{
    width:100%;
  }
</style>
