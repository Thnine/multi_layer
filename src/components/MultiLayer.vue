<template>
  <div ref="multi_layer_container" class="multi_layer_container">
    <svg xmlns="http://www.w3.org/2000/svg" class="multi_layer_canva"></svg>
    <div v-for="(item,index) in innerGraphs" :key="index" ref="multi_layer_controllerBox" :class="`multi_layer_controllerBox-${index}`" style="position:absolute;display:flex;flex-direction:column;">
        <el-popover
          placement="left"
          width="300"
          trigger="click">
          <el-row type="flex" justify="center">
            <b>第{{index+1}}层-设置</b>
          </el-row>
          <hr style="margin-bottom:20px;border:2px solid lightgray;height: 0;"/>
          <el-row type="flex" align="center" style="margin-bottom:20px">
            <el-col :offset="2" :span="10" style="display:flex;align-items:center">点基准半径：</el-col>
            <el-col :offset="0" :span="10">
              <el-input-number 
                size="mini" 
                v-model="baseRadius[index]"
                :min="0.5"
                :step="0.5"
                @change="radiusChange(index)"
                ></el-input-number>
            </el-col>
          </el-row>
          <el-row v-show="index != innerGraphs.length-1" type="flex" align="center">
            <el-col :offset="2" :span="10" style="display:flex;align-items:center">跨层线宽(下层)：</el-col>
            <el-col :offset="0" :span="10">
              <el-input-number 
                size="mini" 
                v-model="outerLinkWidth[index]"
                :min="0.5"
                :step="0.5"
                @change="outLinkWidthChange(index)"
                ></el-input-number>
            </el-col>
          </el-row>
          <el-button slot="reference" type="primary" icon="el-icon-s-tools" circle></el-button>
        </el-popover>
        <el-button type="primary" icon="el-icon-full-screen" circle @click="resetPos(index)" style="margin-top:10px;"></el-button>
    </div>
  </div>
</template>



<script>

import * as d3 from 'd3'
import {nanoid} from 'nanoid'
import {Button,Popover,Row,Col,InputNumber} from 'element-ui';
import Vue from 'vue'
import 'element-ui/lib/theme-chalk/index.css';
import axios from "axios"


Vue.component(Button.name, Button);
Vue.component(Popover.name, Popover);
Vue.component(Row.name, Row);
Vue.component(Col.name, Col);
Vue.component(InputNumber.name, InputNumber);

export default {
  name: 'MultiLayer',
  data () {
    return {
      //根本数据
      nanoid:undefined,//组件的id标识符


      //绘图数据
      innerGraphs:[],//每层的内部数据
      // [
      //   {
      //     'nodes':[{'id':1},...],
      //     'links':[{'source':1,'target':2},...],
      //   },
      //   {
      //     'nodes':[{'id':17},...],
      //     'links':[{'source':23,'target':46},...],
      //   },
      //   ...
      // ]
      // 注意：不同层的nodes的id也必须保持独特性
      outerLinks:[],//层之间的连接数据
      // [
      //   {'links':[{'source':1,'target':17},...],},//source是上层,target是下层
      //   {...},
      //   ...
      // ]


      //临时数据


      //配置项，可人为修改
      outerPadding:{//svg的padding
          'bottom': 40,
          'top': 40,
          'left': 40,
          'right': 120,
      },
      borderSkew:20,//辅助平面的倾斜角度（平面上的倾斜），单位deg
      plotSkew:45,//图像的倾斜角度（空间上的倾斜），单位deg
      WHRadio:0.5,//边框BBox的长宽比
      unitMargin:40,//边框之间的上下间距
      initRadius:6,//点的初始半径
      baseRadius:[],//点的基准半径 [radius1,radius2,...]
      initOuterLinkWidth:3.5,//初始的跨层连线宽度
      outerLinkWidth:[],//跨层连线的宽度
      initInnerPadding:30,//每层的plot在初始的适应状态下相对于initArea的padding（水平方向准，垂直方向不一定准）
      innerArrowSizeRadio:1.7,//层内连线的箭头尺寸对半径的比例
      innerCircleStrokeRadio:0.3,//层内点的stroke-width对半径的比例
      innerLinkStrokeRadio:0.5,//层内的link的宽度对半径的比例


      //计算尺寸
      svgWidth:0,//svg的宽度
      partWidth:0,//边框BBox的宽度
      partHeight:0,//边框的BBox的高度
      borderWidth:0,//边框的宽度
      initAreaWidth:0,//initArea：在折叠前，plot应该占据的空间，initArea的宽度
      initAreaHeight:0,//initArea的高度
    }
  },


  methods:{
    /**
     * 
     * 外部接口
     * 
     */
    setData(innerGraphs,outerLinks){//绑定数据
      //绑定数据
      this.innerGraphs = JSON.parse(JSON.stringify(innerGraphs));
      this.outerLinks = JSON.parse(JSON.stringify(outerLinks));
      console.log('innerGraphs',this.innerGraphs)
      console.log('outerLinks:',this.outerLinks)

      //清理之前的数据，并且初始化一些数据
      this.baseRadius.length = 0;
      for(let i in this.innerGraphs){
        this.baseRadius.push(this.initRadius)
      }
      for(let i in this.outerLinks){
        this.outerLinkWidth.push(this.initOuterLinkWidth);
      }
    },

    draw(){//重绘整张图

      if(this.innerGraphs === undefined || this.innerGraphs === null || this.innerGraphs.length == 0)
        return;

      //清理
      const svg = d3.select(this.$refs['multi_layer_container']).select('.multi_layer_canva');
      svg.selectAll('*').remove();

      //创建绘图结构，并设置层的优先级
      const plot = svg.append('g').classed('multi_layer_plot',true);
      for(let layer in this.innerGraphs){
        //从后往前添加
        const layerArea = plot.append('g').classed(`multi_layer_layerArea-${this.innerGraphs.length - 1 - layer}`,true)
        layerArea.append('g').classed(`multi_layer_outerArea-${this.innerGraphs.length - 1 - layer}`,true)
        layerArea.append('g').classed(`multi_layer_borderArea-${this.innerGraphs.length - 1 - layer}`,true)
        layerArea.append('g').classed(`multi_layer_innerArea-${this.innerGraphs.length - 1 - layer}`,true)
        layerArea.append('defs').classed(`multi_layer_innerDefs-${this.innerGraphs.length - 1 - layer}`,true)
      }

      //适应容器大小
      this.fitSize();

      for(let layer in this.innerGraphs){
        this.drawSingleLayer(parseInt(layer));
      }

      for(let start_layer in this.outerLinks){
        this.drawSingleOuterLinks(parseInt(start_layer));
      }

      
    },


    /**
     * 
     * 内部方法
     * 
     */

    fitSize(){//使得组件适应容器大小（计算单元的各个尺寸，元件位置，以及svg的高度）
      const svg = d3.select(this.$refs['multi_layer_container']).select('.multi_layer_canva');

      //设置长度
      this.svgWidth = svg.node().getBoundingClientRect().width;
      this.partWidth = this.svgWidth - this.outerPadding.left - this.outerPadding.right;
      this.partHeight = this.WHRadio * this.partWidth;
      this.borderWidth = this.partWidth - this.partHeight * Math.tan(this.borderSkew * Math.PI / 180);
      this.initAreaWidth =  this.partWidth - 2 * this.partHeight * Math.tan(this.borderSkew * Math.PI / 180);
      this.initAreaHeight = this.partHeight / Math.cos(this.plotSkew * Math.PI / 180);


      //设置svg高度
      svg.style('height',this.outerPadding.top + this.outerPadding.bottom + this.innerGraphs.length * (this.partHeight + this.unitMargin))
      
      //调整控制器位置
      this.$nextTick(()=>{//防止v-for的渲染延迟
        for(let layer_index in this.innerGraphs){
          const controllerBox = d3.select(this.$refs['multi_layer_container']).select(`.multi_layer_controllerBox-${layer_index}`)
          controllerBox.style('top',`${this.outerPadding.top + layer_index * (this.partHeight + this.unitMargin)}px`)
                       .style('right','20px')
        }
      })

    },

    setLayout(nodes,links,layer_index){//设定图布局，返回恰好适应边框的图布局，布局的左上角(0,0)在画布中对应borderAnchor
      let _nodes = JSON.parse(JSON.stringify(nodes));
      let _links = JSON.parse(JSON.stringify(links));


      /**
       * 
       * 力导引布局
       * 
       * 要求:无要求
       * 
       */
      let simulation = d3.forceSimulation()
                          .nodes(_nodes)


      let linkForce = d3.forceLink(_links).id(d=>d.id)

      simulation.force('charge_force', d3.forceManyBody().strength(-50))
                .force('center_force', d3.forceCenter(0,0))//中心点为原点
                .force('links', linkForce)

      simulation.stop();
      simulation.tick(300);



      /**
       * 
       * 变换坐标到中心为(0.5*initAreaWidth,0.5*initAreaHeight)，边界适应边框。
       * 
       */
      const maxX = Math.max(..._nodes.map(v=>v.x))
      const minX = Math.min(..._nodes.map(v=>v.x))
      const maxY = Math.max(..._nodes.map(v=>v.y))
      const minY = Math.min(..._nodes.map(v=>v.y))
      //变换
      const xScale = d3.scaleLinear()
        .domain([minX,maxX])
        .range([this.baseRadius[layer_index] + this.initInnerPadding,this.initAreaWidth - this.baseRadius[layer_index] - this.initInnerPadding])
      const yScale = d3.scaleLinear()
        .domain([minY,maxY])
        .range([this.baseRadius[layer_index] + this.initInnerPadding,this.initAreaHeight - this.baseRadius[layer_index] - this.initInnerPadding])
      
      _nodes.forEach((v,i)=>{
        _nodes[i].x = xScale(_nodes[i].x)
        _nodes[i].y = yScale(_nodes[i].y)
      })


      /**
       * 
       * 打包布局结果
       * 
       */
      let result = {
        'nodes':_nodes.map(v=>{
                  return {
                    'id':v.id,
                    'x':v.x,
                    'y':v.y,
                  }
                }),
        'links':_links.map(v=>{
                  return {
                    'source':{'id':v.source.id,'x':v.source.x,'y':v.source.y},
                    'target':{'id':v.target.id,'x':v.target.x,'y':v.target.y},
                  }
                }),
      }

      return result
                               
    },


    getWangZixiaoLayout(){
      /**
       * 
       * 王子潇布局算法
       * 
       * 要求:1.跨层有连接关系的点必须是一对一的关系
       *      2.下层点多于上层点
       * 
       */
      let layer_data = ['layerID layerLabel']
      let layer_map = []
      let node_data = ['nodeID nodeAge nodeTenure nodeLevel nodeDepartment']
      let edge_data = []
      for(let layer_index in this.innerGraphs){
        layer_data.push(`${parseInt(layer_index)+1} layer_name`);//构建layer data
        //初始化layer_map
        layer_map.push({})  
      }
      let whole_id_set = new Set()
      this.innerGraphs[this.innerGraphs.length-1].nodes.forEach((v,i)=>{
        layer_map[this.innerGraphs.length-1][String(v.id)] = (i + 1)
        whole_id_set.add((i + 1))
        node_data.push(`${(i + 1)} 0 0 0 0`)
      })
      for(let layer_index in this.outerLinks){
        //倒序给layer_map赋值
        let ol = this.outerLinks[this.outerLinks.length - parseInt(layer_index) - 1].links;
        let upper = layer_map[this.outerLinks.length - parseInt(layer_index) - 1];//上层
        let lower = layer_map[this.outerLinks.length - parseInt(layer_index)];//下层
        
        let new_id_set = new Set(whole_id_set) //保存了所有的布局用id
        let raw_upper_id_queue = this.innerGraphs[this.outerLinks.length - parseInt(layer_index) - 1].nodes.map(v=>v.id)//保存了上层的innerGrah中的id
        ol.forEach((v,i)=>{//利用边建立关系
          upper[String(v.source)] = lower[String(v.target)]
          new_id_set.delete(lower[String(v.target)])
          raw_upper_id_queue.splice(raw_upper_id_queue.indexOf(parseInt(v.source)),1)
        })
        //给set中的剩余项赋值
        for(let remain_id of Array.from(new_id_set)){
          if(raw_upper_id_queue.length != 0){//raw_upper_id_queue还有剩余
            upper[String(raw_upper_id_queue.pop())] = remain_id;
          }
          else{//raw_upper_id_queue无剩余
            upper[String(remain_id)] = remain_id;
          }
        }
      }
      //添加edge_data
      this.innerGraphs.forEach((v,layer_index)=>{
        let links = v.links;
        links.forEach((l)=>{
            edge_data.push(`${layer_index+1} ${layer_map[layer_index][String(l.source)]} ${layer_map[layer_index][String(l.target)]} 1`)
        })
        
      })
      
      console.log('layer_map:',JSON.parse(JSON.stringify(layer_map)))
      console.log('layer_data:',layer_data)
      console.log('node_data:',node_data)
      console.log('edge_data:',edge_data)

    },

    getForceLayout(nodes,links){
      for(let layer_index in this.innerGraphs){
          let _nodes = JSON.parse(JSON.stringify(nodes));
          let _links = JSON.parse(JSON.stringify(links));


          /**
           * 
           * 力导引布局
           * 
           * 要求:无要求
           * 
           */
          let simulation = d3.forceSimulation()
                              .nodes(_nodes)


          let linkForce = d3.forceLink(_links).id(d=>d.id)

          simulation.force('charge_force', d3.forceManyBody().strength(-50))
                    .force('center_force', d3.forceCenter(0,0))//中心点为原点
                    .force('links', linkForce)

          simulation.stop();
          simulation.tick(300);



          /**
           * 
           * 变换坐标到中心为(0.5*initAreaWidth,0.5*initAreaHeight)，边界适应边框。
           * 
           */
          const maxX = Math.max(..._nodes.map(v=>v.x))
          const minX = Math.min(..._nodes.map(v=>v.x))
          const maxY = Math.max(..._nodes.map(v=>v.y))
          const minY = Math.min(..._nodes.map(v=>v.y))
          //变换
          const xScale = d3.scaleLinear()
            .domain([minX,maxX])
            .range([this.baseRadius[layer_index] + this.initInnerPadding,this.initAreaWidth - this.baseRadius[layer_index] - this.initInnerPadding])
          const yScale = d3.scaleLinear()
            .domain([minY,maxY])
            .range([this.baseRadius[layer_index] + this.initInnerPadding,this.initAreaHeight - this.baseRadius[layer_index] - this.initInnerPadding])
          
          _nodes.forEach((v,i)=>{
            _nodes[i].x = xScale(_nodes[i].x)
            _nodes[i].y = yScale(_nodes[i].y)
          })


          /**
           * 
           * 打包布局结果
           * 
           */
          let result = {
            'nodes':_nodes.map(v=>{
                      return {
                        'id':v.id,
                        'x':v.x,
                        'y':v.y,
                      }
                    }),
            'links':_links.map(v=>{
                      return {
                        'source':{'id':v.source.id,'x':v.source.x,'y':v.source.y},
                        'target':{'id':v.target.id,'x':v.target.x,'y':v.target.y},
                      }
                    }),
          }

          return result 
      }
    },


    drawSingleLayer(layer_index){//绘制/更新单层
        /**
         * params：
         *    layer_index:平面的层次，
         * 
         */
        const self = this;

        const svg = d3.select(this.$refs['multi_layer_container']).select('.multi_layer_canva');
        const layerArea = svg.select(`.multi_layer_layerArea-${layer_index}`)
        const innerArea = layerArea.select(`.multi_layer_innerArea-${layer_index}`)
        const borderArea = layerArea.select(`.multi_layer_borderArea-${layer_index}`)
        const innerDefs = layerArea.select(`.multi_layer_innerDefs-${layer_index}`)

        //清理
        innerArea.selectAll('*').remove();
        borderArea.selectAll('*').remove();
        innerDefs.selectAll('*').remove();


        const anchor = [
          this.outerPadding.left,
          this.outerPadding.top + layer_index * (this.partHeight + this.unitMargin)
        ]//边框BBox左上角的点坐标
        const borderAnchor = [
          this.outerPadding.left + this.partHeight * Math.tan(this.borderSkew * Math.PI / 180),
          this.outerPadding.top + layer_index * (this.partHeight + this.unitMargin)
        ]//边框左上角的点坐标


        /**
         * 
         * 绘制本层边框
         * 
         */
        const border = borderArea.append('path')
            .attr('d', `M ${borderAnchor[0]} ${borderAnchor[1]} `
                      +`L ${borderAnchor[0] + this.borderWidth} ${borderAnchor[1]} ` 
                      +`L ${anchor[0] + this.borderWidth} ${anchor[1] + this.partHeight} `
                      +`L ${anchor[0]} ${anchor[1] + this.partHeight} `
                      +`L ${borderAnchor[0]} ${borderAnchor[1]}`)
            .attr('fill', '#1f8bd4')
            .attr('fill-opacity', 0.7)
            .style('stroke', "lightgray")
            .style('stroke-width', 3)
            .classed(`multi_layer-border-${layer_index}`,true)
        
        /**
         * 
         * 绘制层内图（点和边）
         * 
         */
        let nodes = this.innerGraphs[layer_index].nodes;
        let innerLinks = this.innerGraphs[layer_index].links;
        let layoutResult = this.setLayout(nodes,innerLinks,layer_index);
        let nodesPos = layoutResult['nodes']
        let innerLinksPos = layoutResult['links']


        //定义遮罩
        innerDefs.append('clipPath').attr("id",`multi_layer_clip-${layer_index}-${this.nanoid}`)//使用nanoid是为了组件被复用的时候重复
            .append('path')
            .attr('d', `M ${borderAnchor[0]} ${borderAnchor[1]} `
                      +`L ${borderAnchor[0] + this.borderWidth} ${borderAnchor[1]} ` 
                      +`L ${anchor[0] + this.borderWidth} ${anchor[1] + this.partHeight} `
                      +`L ${anchor[0]} ${anchor[1] + this.partHeight} `
                      +`L ${borderAnchor[0]} ${borderAnchor[1]}`)
        //定义arrow
        let innerArrowSize = this.innerArrowSizeRadio * this.baseRadius[layer_index]
        innerDefs.append('marker')
                .attr('id', `multi_layer_inner_arrow-${layer_index}-${this.nanoid}`)
                .attr('markerWidth', innerArrowSize)
                .attr('markerHeight', innerArrowSize)
                .attr('refX', innerArrowSize + this.baseRadius[layer_index])
                .attr('refY', 0.5 * innerArrowSize)
                .attr("markerUnits","userSpaceOnUse")
                .attr('orient', 'auto')
                .append('path')
                .attr('fill', '#434343')
                .attr('d', `M 0,0 L ${innerArrowSize},${0.5*innerArrowSize} L 0,${innerArrowSize}`)
                // .attr('d', 'M 0,0 L 10,5 L 0,10')

        //添加范围遮罩（用于裁剪）
        const clipG = innerArea.append('g').attr('clip-path',`url(#multi_layer_clip-${layer_index}-${this.nanoid})`)
        //添加zoom层（用于被zoom位移）
        const zoomG = clipG.append('g')
        //添加innerPlot层（用于容纳图元、倾斜）
        const innerPlot = zoomG.append('g')
                               .classed(`multi_layer_innerPlot-${layer_index}`,true)
                               
        innerPlot.append('g')
          .selectAll('*')
          .data(innerLinksPos)
          .join('line')
          .attr('stroke-width', this.baseRadius[layer_index] * this.innerLinkStrokeRadio)
          .style('stroke', "#666666")
          .attr('marker-end',`url(#multi_layer_inner_arrow-${layer_index}-${this.nanoid})`)
          .attr('x1', (d) => d.source.x + borderAnchor[0])
          .attr('y1', (d) => d.source.y + borderAnchor[1])
          .attr('x2', (d) => d.target.x + borderAnchor[0])
          .attr('y2', (d) => d.target.y + borderAnchor[1])
          .classed(`multi_layer_innerLinks-${layer_index}`,true)
        innerPlot.append('g')
          .selectAll('*')
          .data(nodesPos)
          .join('circle')
          .attr("r",this.baseRadius[layer_index])
          .attr('fill', "#b256f0")
          .attr('stroke', "white")
          .attr('stroke-width', this.innerCircleStrokeRadio * this.baseRadius[layer_index])
          .attr("cx",(d)=>d.x + borderAnchor[0])
          .attr("cy",(d)=>d.y + borderAnchor[1])
          .classed(`multi_layer_innerCircles-${layer_index}`,true)
          .style('cursor','pointer')
          .on('click',(d,i)=>{
            console.log(d.id)
          })

        //倾斜
        innerPlot.style('transform-origin',`${borderAnchor[0]}px ${borderAnchor[1]}px`)
        innerPlot.style('transform', `rotateX(${this.plotSkew}deg)`)
        
        //zoom的交互操作遮罩
        // const zoomMask = innerArea.append('path')
        //     .attr('d', `M ${borderAnchor[0]} ${borderAnchor[1]} `
        //               +`L ${borderAnchor[0] + this.borderWidth} ${borderAnchor[1]} ` 
        //               +`L ${anchor[0] + this.borderWidth} ${anchor[1] + this.partHeight} `
        //               +`L ${anchor[0]} ${anchor[1] + this.partHeight} `
        //               +`L ${borderAnchor[0]} ${borderAnchor[1]}`)
        //     .attr('fill', '#1f8bd4')
        //     .attr('fill-opacity', 0)

        //绑定zoom事件
        const zoom = d3.zoom()
                       .scaleExtent([0.1, 40])
                       .translateExtent([[-10000, -10000], [10000, 10000]])
                       .on("zoom",()=>{
                          zoomG.attr("transform",d3.event.transform)                                              
                          this.drawSingleOuterLinks(layer_index)
                          this.drawSingleOuterLinks(layer_index-1)
                       })
        border.call(zoom)



      
    },

    drawSingleOuterLinks(layer_index){// 绘制/更新外部的连接边，这个函数必须在drawSingleLayer之后调用
      

      const self = this;

      //判断异常
      if(layer_index < 0 || layer_index >= this.outerLinks.length)
        return;

      let drawLinks = this.outerLinks[layer_index]['links'];

      const svg = d3.select(this.$refs['multi_layer_container']).select('.multi_layer_canva');
      const layerArea1 = svg.select(`.multi_layer_layerArea-${layer_index}`)
      const layerArea2 = svg.select(`.multi_layer_layerArea-${layer_index+1}`)
      const outerArea = layerArea1.select(`.multi_layer_outerArea-${layer_index}`)
      const border1 = layerArea1.select(`.multi_layer-border-${layer_index}`)
      const border2 = layerArea2.select(`.multi_layer-border-${layer_index+1}`)


      outerArea.selectAll('*').remove();

      const nodes1 = layerArea1.selectAll(`.multi_layer_innerCircles-${layer_index}`)//上层点
      const nodes2 = layerArea2.selectAll(`.multi_layer_innerCircles-${layer_index+1}`)//下层点

      const links = outerArea.append('g')
                  .selectAll('*')
                  .data(drawLinks)
                  .join('line')
                  .attr("x1",function(d){//1为上层点，2为下层点
                      let node1 = nodes1.filter((v, i) => {
                          return d.source == v.id
                      })
                      return node1.node().getBoundingClientRect().x - svg.node().getBoundingClientRect().x + 0.5 * node1.node().getBoundingClientRect().width;
                  })
                  .attr("y1",function(d){//1为上层点，2为下层点
                      let node1 = nodes1.filter((v, i) => {
                          return d.source == v.id
                      })
                      return node1.node().getBoundingClientRect().y - svg.node().getBoundingClientRect().y + 0.5 * node1.node().getBoundingClientRect().height;
                  })
                  .attr("x2",function(d){//1为上层点，2为下层点
                      let node2 = nodes2.filter((v, i) => {
                          return d.target == v.id
                      })
                      return node2.node().getBoundingClientRect().x - svg.node().getBoundingClientRect().x + 0.5 * node2.node().getBoundingClientRect().width;
                  })
                  .attr("y2",function(d){//1为上层点，2为下层点
                      let node2 = nodes2.filter((v, i) => {
                          return d.target == v.id
                      })
                      return node2.node().getBoundingClientRect().y - svg.node().getBoundingClientRect().y + 0.5 * node2.node().getBoundingClientRect().height;
                  })
                  .attr('stroke', '#434343')
                  .attr('stroke-width', this.outerLinkWidth[layer_index])
                  .attr('opacity', 0.7)
                  .style('display',(d,i)=>{
                      //node1为上层点 //node2为下层点
                      let node1 = nodes1.filter((v, i) => {
                          return d.source == v.id
                      })
                      let node2 = nodes2.filter((v, i) => {
                          return d.target == v.id
                      })
                      //检测端点是否溢出
                      const point1 = svg.node().createSVGPoint();
                      point1.x = node1.node().getBoundingClientRect().x - svg.node().getBoundingClientRect().x + 0.5 * node1.node().getBoundingClientRect().width;
                      point1.y = node1.node().getBoundingClientRect().y - svg.node().getBoundingClientRect().y + 0.5 * node1.node().getBoundingClientRect().height
                      const point2 = svg.node().createSVGPoint();
                      point2.x = node2.node().getBoundingClientRect().x - svg.node().getBoundingClientRect().x + 0.5 * node2.node().getBoundingClientRect().width;
                      point2.y = node2.node().getBoundingClientRect().y - svg.node().getBoundingClientRect().y + 0.5 * node2.node().getBoundingClientRect().height

                      if(!border1.node().isPointInFill(point1) && !border1.node().isPointInStroke(point1))
                        return 'none'

                      if(!border2.node().isPointInFill(point2) && !border2.node().isPointInStroke(point2))
                        return 'none'
                      
                      return null
                  })      

    },
    
    radiusChange(layer_index){//某一层的radius被修改的回调函数
      const svg = d3.select(this.$refs['multi_layer_container']).select('.multi_layer_canva');
      const layerArea = svg.select(`.multi_layer_layerArea-${layer_index}`)
      const innerArea = layerArea.select(`.multi_layer_innerArea-${layer_index}`)
      const borderArea = layerArea.select(`.multi_layer_borderArea-${layer_index}`)
      const innerDefs = layerArea.select(`.multi_layer_innerDefs-${layer_index}`)


      //修改defs的marker
      let innerArrowSize = this.innerArrowSizeRadio * this.baseRadius[layer_index]
      const markerDefs = innerDefs.select(`#multi_layer_inner_arrow-${layer_index}-${this.nanoid}`)
      markerDefs.attr('markerWidth', innerArrowSize)
                .attr('markerHeight', innerArrowSize)
                .attr('refX', innerArrowSize + this.baseRadius[layer_index])
                .attr('refY', 0.5 * innerArrowSize)


      markerDefs.select('path')
                .attr('d', `M 0,0 L ${innerArrowSize},${0.5*innerArrowSize} L 0,${innerArrowSize}`)


      //修改层内点的半径和stroke-width
      const innerCircles = layerArea.selectAll(`.multi_layer_innerCircles-${layer_index}`);
      innerCircles.attr("r",this.baseRadius[layer_index])
                  .attr('stroke-width', this.innerCircleStrokeRadio * this.baseRadius[layer_index])


      //修改连接线的stroke-width
      const innerLinks = layerArea.selectAll(`.multi_layer_innerLinks-${layer_index}`);
      innerLinks.attr('stroke-width', this.baseRadius[layer_index] * this.innerLinkStrokeRadio)
    },

    resetPos(layer_index){//对某层进行复位
      this.drawSingleLayer(layer_index);
      this.drawSingleOuterLinks(layer_index-1);
      this.drawSingleOuterLinks(layer_index)
    },
    outLinkWidthChange(layer_index){//修改某层OutLinks的宽度
      this.drawSingleOuterLinks(layer_index-1);
      this.drawSingleOuterLinks(layer_index)
    }


  },




  mounted(){
    this.nanoid = nanoid();

  }

}
</script>



<style scoped>
  /* 容器样式 */
  .multi_layer_container{
    height:100%;
    width: 100%;
    overflow: auto;
    display: flex;
    position:relative
  }
  .multi_layer_canva{
    width:100%;
  }

  /* 动态样式 */
  .multi_layer_circle_chosen{

  }


</style>
