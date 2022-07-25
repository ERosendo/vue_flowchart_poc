<template>

  <div class="d-flex flex-row h-100" >
    <div class="d-flex flex-column m-2 col-lg-3" style='border:1px solid black;'>
      <p class='w-100 m-1 p-1'>Lista de Estados</p>
      <div class="d-flex flex-column h-100" style="overflow-y:scroll;">
        <div v-for="s in status" :key="s.id" class="d-flex p-2 m-2 justify-content-between" style='border:1px solid black;'>
          {{ s.name }}
          <button @click="addNode(s)" :disabled='s.disable'>Añadir Nodo</button>
        </div>
      </div>
      <div v-if='!createStatus' class='m-2 p-2'>
        <button @click='createStatus=true'>Crear nuevo Estado</button>
      </div>
      <div v-if='createStatus' class='m-2 p-2' >
        <div class='m-2'>
          <div class='m-2'>
            Formulario de creacion
          </div>
          <div class="form-group d-flex justify-content-between m-1">
            <label for="Name">Name</label>
            <input type="text" id="name" v-model="statusForm.name">
          </div>
          <div class='form-group d-flex justify-content-between m-1'>
            <label for="description">Description</label>
            <input type="text" id='description' v-model="statusForm.description">
          </div>
        </div>
        <div>
          <button @click="sendForm">Enviar</button>
          <button @click="cancelForm">Cancelar</button>
        </div>
      </div>
    </div>
    <div class= 'd-flex m-2 col-lg-9' style="flex: 1 1 auto;">
      <VueFlow 
        class="basicflow" 
        :default-zoom="1" 
        :min-zoom="0.2" 
        :max-zoom="4"
        @nodeClick="nodeClick"
        @edgeClick="edgeClick"
        @connect="onConnect"
        :fit-view-on-init="true"
        >
        <Controls />
        <div class="controls">
          <button @click="removeNode" v-show="selectedId">Remove Node</button>
          <button @click="removeEdge" v-show="selectedEdgeId">Remove Edge</button>
          <button @click="saveNodes">Save</button>
        </div>
      </VueFlow>
    </div>
  </div>
</template>



<script>
import { ref } from "vue";
import { VueFlow, useVueFlow, Controls } from "@braks/vue-flow";

import axios from "axios";

export default {
  name: "vue-flowchart",

  components: {
    VueFlow,
    Controls
  },

  setup() {
    let status = ref([])
    let selectedId = ref(null);
    let createStatus = ref(false);
    let selectedEdgeId = ref(null);
    let statusForm = ref({
      name: null,
      description: null
    })
    let requestURL = ref(process.env.VUE_APP_FLOWCHAR_API)
    let requestId = ref(process.env.VUE_APP_WORKFLOW_ID)
    let typeId = ref(process.env.VUE_APP_STATUS_TYPE_ID)

    const { nodes, edges ,dimensions, getNode, addEdges, addNodes, applyNodeChanges, applyEdgeChanges} = useVueFlow()
    
    return {
      requestURL,
      createStatus,
      statusForm,
      typeId,
      status,
      nodes,
      edges,
      dimensions,
      selectedId,
      selectedEdgeId,
      getNode,
      requestId,
      addNodes,
      addEdges,
      applyNodeChanges,
      applyEdgeChanges
    };
  },

  async mounted() {
    this.getStatus()
    await this.getNodes();
    await this.getEdges();
  },

  methods: {
    async getEdges() {
      await axios
        .get(
          `${this.requestURL}/workflows/${this.requestId}/transitions/`
        )
        .then((response) => {
          let data = response.data
          for (let i=0; i< data.length; i++){
            let newEdge = {
              id: data[i].id,
              source: data[i].source.toString(),
              target: data[i].target.toString(),
              sourceHandle: null,
              targetHandle: null,
            }
            this.addEdges([newEdge])
          }
        });
    },

    onLoad(flowInstance){
      flowInstance.fitView()
    },

    getStatus(){
      axios
        .get(`${this.requestURL}/status/`, { params: { workflow_id: this.requestId } })
        .then((request)=>{
          this.status = request.data
        })
    },

    async getNodes() {
      await axios
        .get(
          `${this.requestURL}/workflows/${this.requestId}/status/`
        )
        .then((response) => {
          let data = response.data;
          for (let i = 0; i< data.length; i++){
            let newNode = {
              id: data[i].id,
              label: data[i].label,
              status_id: data[i].status_id,
            }

            if (!Object.keys(data[i].position).length){
              newNode['position'] = { 
                x: Math.random() * this.dimensions.width, 
                y: Math.random() * this.dimensions.height 
              }
            }else{
              newNode['position'] = data[i].position
            }
           this.addNodes([newNode])
          }
        });
    },

    addNode(node){
      axios
        .post(`${this.requestURL}/workflows/add_status/`, {
          status_id : node.id,
          workflow_id: this.requestId
        })
        .then((response)=>{
            let data = response.data
            const newNode = {
              id: data.id,
              label: data.label,
              status_id: data.status_id,
              position: { 
                x: Math.random() * this.dimensions.width, 
                y: Math.random() * this.dimensions.height 
              },
            }
            this.addNodes([newNode])

            let index = this.status.findIndex(x => x.id === data.status_id);
            let selectedStatus = this.status[index]
            selectedStatus.disable = true 
          }
        )      
    },

    nodeClick(NodeMouseEvent){
      this.selectedEdgeId = null 

      if (this.selectedId){
        const previous_selected_node =  this.getNode(this.selectedId)
        delete previous_selected_node.style
      }
      
      this.selectedId = NodeMouseEvent.node.id
    
      const new_selected_node = this.getNode(this.selectedId)
      new_selected_node.style = { background: '#D6D5E6', color: '#333', border: '1px solid #222138', width: 180 }
      
    },

    edgeClick({edge}){
      this.selectedId = null
      this.selectedEdgeId = edge.id
    },

    async saveNodes(){
      for ( let i=0; i<this.nodes.length ;i++){
        let node_id = this.nodes[i].id.toString()
        let temp_node = this.getNode(node_id)
        await axios
          .patch(
            `${this.requestURL}/status_workflow/${node_id}/`,
            {
              position: {
                x: temp_node.position.x,
                y: temp_node.position.y
              },
            }
          )
          .then((response) => {
            console.log(response)
          });
      }
    },

    async onConnect(params){
      await axios
        .post(`${this.requestURL}/transitions/`,
        {
          workflow_id: this.requestId,
          current_status_id: params.source,
          next_status_id: params.target
        })
        .then((response)=>{
          let data = response.data

          let newEdge = {
            id: data.id,
            source: data.current_status_id.toString(),
            target: data.next_status_id.toString(),
            sourceHandle: null,
            targetHandle: null,
          }

          this.addEdges([newEdge])
        })
    },

    removeEdge(){
      axios
        .delete(`${this.requestURL}/transitions/${this.selectedEdgeId}/`)
        .then(()=>{
          this.applyEdgeChanges([
            {
              id: this.selectedEdgeId,
              type: 'remove',
            }
          ])
          this.selectedEdgeId = null
        })
    },

    sendForm(){
      axios
        .post(`${this.requestURL}/status/`,{
          name: this.statusForm.name,
          description: this.statusForm.description,
          type_id: this.typeId
        })
        .then((response)=>{
          let data = response.data
          this.status.push(data)

          this.createStatus = false

          this.statusForm = {
            name: null,
            description: null
          }
        })
    },

    cancelForm(){
      this.createStatus=false
      this.statusForm = {
        name: null,
        description: null
      }
    },

    removeNode(){
      axios
        .delete(`${this.requestURL}/status_workflow/${this.selectedId}/`)
        .then(()=>{
          let index = this.nodes.findIndex(x => x.id == parseInt(this.selectedId));
          let statusId = this.nodes[index].status_id

          this.applyNodeChanges([
            {
              id: this.selectedId,
              type: 'remove',
            }
          ])
          
          let statusIndex = this.status.findIndex(x => x.id == statusId);
          let selectedStatus = this.status[statusIndex]
          selectedStatus.disable = false
          this.selectedId = null
        })
    },
  },
};
</script>

<style scoped>
  .popup {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    z-index: 99;
    background-color: rgba(0, 0, 0, 0.2);
    
    display: flex;
    align-items: center;
    justify-content: center;
  }
</style>

