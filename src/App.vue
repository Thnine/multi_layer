<template>
  <div id="app">
    <div style="width:800px;height:1000px">
      <MultiLayer @exportChosenData="handleExportChoseData" style="margin:70px 40px" ref="MultiLayer"/>
    </div>
  </div>
</template>

<script>
import MultiLayer from './components/MultiLayer/MultiLayer'
import * as d3 from 'd3';
import {groupSort,group,sort} from 'd3-array'
export default {
  name: 'App',
  components: {
    MultiLayer
  },
  methods:{
    handleExportChoseData(val){
      console.log('chosen:',val)
    }
  },
  mounted(){
      //nodes : [{'id':'1',layer:1,...},{'id':'2',layer:0,...},...] 要求符合标准json格式，layer要求为数字，不要求连续，数字小的绘图会在上层
      //links : [{'source':id3,'target':id41,...},...] 要求符合标准json格式，对于跨层边，source要求是上层点的index，target要求是下层点的index
    //   d3.json('static/links.json').then(links=>{
    //     d3.json('static/nodes.json').then(nodes=>{
    //       console.log('links:',links)
    //       console.log('nodes:',nodes)
    //       //处理数据
    //       let outerLinks = []
    //       let innerGraphs = []
    //       let node_to_layer_map = new Map()
    //       for(let n of nodes){
    //         node_to_layer_map.set(n['index'],n['layer'])
    //       }
    //       let node_groups = Array.from(group(nodes,d=>d.layer));
    //       node_groups = sort(node_groups,(a,b)=>{
    //         return parseInt(a[0])-parseInt(b[0])
    //       });
    //       node_groups.forEach((n,i)=>{
    //         let iGraph = {
    //           'nodes':n[1].map(v=>{
    //             let obj = {}
    //             for(let key in v){
    //               if(key == 'index')
    //                 obj['id'] = v[key]
    //               else
    //                 obj[key] = v[key]
    //             }
    //             return obj
    //           }),
    //           'links':[],
    //         }
            
    //         //获取innerLinks
    //         for(let l of links){
    //           if(node_to_layer_map.get(l[0])==i && node_to_layer_map.get(l[1])==i){
    //             iGraph.links.push({
    //               'source':l[0],
    //               'target':l[1],
    //               'type':l[2],
    //             })
    //           }
    //         }
    //         innerGraphs.push(iGraph)
    //       })
    //       for(let i = 0;i < innerGraphs.length-1;i++){
    //         outerLinks.push({
    //           'links':[]
    //         })
    //       }
    //       //获取outerLinks
    //       for(let l of links){
    //         if(node_to_layer_map.get(l[0]) != node_to_layer_map.get(l[1])){
    //           outerLinks[Math.min(node_to_layer_map.get(l[0]),node_to_layer_map.get(l[1]))].links.push({
    //             'source':l[0],
    //             'target':l[1],
    //             'type':'undir',
    //           })
    //         }
    //       }

    //       //整理message
    //       for(let i = 0;i < innerGraphs.length;i++){
    //           let iGraph = innerGraphs[i];         
    //           for(let j = 0;j < iGraph.nodes.length;j++){
    //             let message = {}
    //             for(let key in iGraph.nodes[j]){
    //               message[key] = iGraph.nodes[j][key];
    //             }
    //             iGraph.nodes[j].message = message
    //           }
    //           for(let j = 0;j < iGraph.links.length;j++){
    //             let message = {}
    //             for(let key in iGraph.links[j]){
    //               message[key] = iGraph.links[j][key];
    //             }
    //             iGraph.links[j].message = message
    //           }
    //           if(i < innerGraphs.length - 1){
    //             for(let j = 0;j < outerLinks[i].links.length;j++){
    //               let message = {}
    //               for(let key in outerLinks[i].links[j]){
    //                 message[key] = outerLinks[i].links[j][key];
    //               }
    //               outerLinks[i].links[j].message = message
    //             }
    //           }
    //       }

    //       this.$refs['MultiLayer'].setData(innerGraphs,outerLinks)
    //       this.$refs['MultiLayer'].draw()
    //   })
    // })
  
    function convertData(json_data){
          let nodes = json_data['graph_nodes']
          let links = json_data['graph_edges']
          console.log('links:',links)
          console.log('nodes:',nodes)
          //处理数据
          let outerLinks = []
          let innerGraphs = []
          let node_to_layer_map = new Map()
          for(let n of nodes){
            node_to_layer_map.set(n['id'],n['layer'])
          }
          let node_groups = Array.from(group(nodes,d=>d.layer));
          node_groups = sort(node_groups,(a,b)=>{
            return parseInt(a[0])-parseInt(b[0])
          });
          node_groups.forEach((n,i)=>{
            let iGraph = {
              'nodes':n[1].map(v=>{
                let obj = {}
                for(let key in v){
                  if(key == 'id')
                    obj['id'] = v[key]
                  else
                    obj[key] = v[key]
                }
                return obj
              }),
              'links':[],
            }
            
            //获取innerLinks
            for(let l of links){
              if(node_to_layer_map.get(l.source)==i && node_to_layer_map.get(l.target)==i){
                iGraph.links.push({
                  'source':l.source,
                  'target':l.target,
                })
              }
            }
            innerGraphs.push(iGraph)
          })
          for(let i = 0;i < innerGraphs.length-1;i++){
            outerLinks.push({
              'links':[]
            })
          }
          //获取outerLinks
          for(let l of links){
            if(node_to_layer_map.get(l.source) != node_to_layer_map.get(l.target)){
              outerLinks[Math.min(node_to_layer_map.get(l.source),node_to_layer_map.get(l.target))].links.push({
                'source':l.source,
                'target':l.target,
              })
            }
          }

          //整理message
          for(let i = 0;i < innerGraphs.length;i++){
              let iGraph = innerGraphs[i];         
              for(let j = 0;j < iGraph.nodes.length;j++){
                let message = {}
                for(let key in iGraph.nodes[j]){
                  message[key] = iGraph.nodes[j][key];
                }
                iGraph.nodes[j].message = message
              }
              for(let j = 0;j < iGraph.links.length;j++){
                let message = {}
                for(let key in iGraph.links[j]){
                  message[key] = iGraph.links[j][key];
                }
                iGraph.links[j].message = message
              }
              if(i < innerGraphs.length - 1){
                for(let j = 0;j < outerLinks[i].links.length;j++){
                  let message = {}
                  for(let key in outerLinks[i].links[j]){
                    message[key] = outerLinks[i].links[j][key];
                  }
                  outerLinks[i].links[j].message = message
                }
              }
          }

          return {
            'innerGraphs':innerGraphs,
            'outerLinks':outerLinks,
          }
    }

    d3.json('static/multi_layer(up-less).json').then((json_data)=>{
      let data = convertData(json_data)
      this.$refs['MultiLayer'].setData(data.innerGraphs,data.outerLinks)
      this.$refs['MultiLayer'].draw()
    })

    

  }
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
