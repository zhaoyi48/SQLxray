<template>

    <VueFlow v-model="elements">
        <MiniMap />
        <Controls :showInteractive="false" />
        <Background :variant="BackgroundVariant.Lines" />
    </VueFlow>

</template>

<script setup>
import { Background, BackgroundVariant, Controls, MiniMap, VueFlow, useVueFlow } from '@braks/vue-flow'
import { provide,inject, ref, shallowRef } from 'vue'

const elements = inject('map')

const { nodes, addNodes, edges, addEdges, onConnect, onPaneReady, onNodeDragStop, dimensions, KeyCode } = useVueFlow()

onConnect((params) => addEdges([params]))
onPaneReady((flowInstance) => console.log('flow loaded:', flowInstance))
onNodeDragStop((node) => console.log('drag stop', node))

// const addRandomNode = () => {
//     const nodeId = (nodes.value.length + 1).toString()
//     const newNode = {
//         id: nodeId,
//         label: `Node: ${nodeId}`,
//         position: { x: Math.random() * dimensions.value.width, y: Math.random() * dimensions.value.height },
//     }
//     addNodes([newNode])
// }

</script>