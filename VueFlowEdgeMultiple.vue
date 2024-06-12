<script setup lang="ts">
import { Edge, EdgeLabelRenderer, getBezierPath } from '@vue-flow/core';
import { PropType } from 'vue';

const props = defineProps({
  id: {
    type: String,
    required: true,
  },
  source: {
    type: String,
    required: true,
  },
  target: {
    type: String,
    required: true,
  },
  sourceX: {
    type: Number,
    required: true,
  },
  sourceY: {
    type: Number,
    required: true,
  },
  targetX: {
    type: Number,
    required: true,
  },
  targetY: {
    type: Number,
    required: true,
  },
  sourcePosition: {
    types: [String, undefined],
    required: false,
    default: undefined,
  },
  targetPosition: {
    types: [String, undefined],
    required: false,
    default: undefined,
  },
  data: {
    type: Object as PropType<{ [key: string]: any }>,
    required: false,
    default: () => ({}),
  },
  markerEnd: {
    type: String,
    required: false,
  },
  style: {
    type: Object,
    required: false,
  },

  // Custom props
  relatedEdges: {
    type: Array as PropType<Edge[]>,
    required: true,
  },
  position: {
    type: Number,
    required: true,
  },
})

const state = reactive({
  labelRotation: 0,
  edgeOffset: 0,
  isHorizontal: false,
})

const siblingsAmount = computed(() => {
  return props.relatedEdges.edges.length + 1
})

const label = computed(() => {
  const label = props.data.label
  const distance = Math.abs(props.sourceX - props.targetX) + Math.abs(props.sourceY - props.targetY)
  const minimumLength = 5
  
  const availableChars = Math.sqrt(distance) + minimumLength
  return `${label.slice(0, availableChars)}${label.length > availableChars ? '...' : ''})`
})

const labelOffset = computed(() => {
  if (siblingsAmount.value === 1) {
    return '0px, -10px'
  }
  
  const position = siblingsAmount.value
  const offsetTextRatio = 0.75
  let offsetY = position % 2 === 0 ? (position / 2) * 0 : -((position + 1) / 2) * 6
  let offsetX = 0

  if (state.isHorizontal) {
    offsetY = offsetY + state.edgeOffset * offsetTextRatio
  } else {
    offsetX = offsetX + state.edgeOffset * offsetTextRatio
  }

  return `${offsetX}px, ${offsetY}px`
})

const path = computed(() => {
  const position = props.position
  const basePath = getBezierPath({
    sourceX: props.sourceX,
    sourceY: props.sourceY,
    sourcePosition: props.sourcePosition,
    targetX: props.targetX,
    targetY: props.targetY,
    targetPosition: props.targetPosition,
  })

  if (siblingsAmount.value === 1) {
    return basePath
  }

  else {
    // exemple of basePath: ['M75,400 C75,400 515,400 515,400', 295, 400, 220, 0]
    const [path, x1, y1, x2, y2] = basePath
    const newPath = generateOffsetedPath({siblingsAmount: siblingsAmount.value, position, path, x1, y1, x2, y2})
    return [newPath, x1, y1, x2, y2]
  }
})

function generateOffsetedPath(params: {siblingsAmount: number, position: number, path: string, x1: number, y1: number, x2: number, y2: number}) {
  const [startX, startY, control1X, control1Y, control2X, control2Y, endX, endY] = params.path.split(' ').map((a: string) => a.split(',').map((b: string) => b.match(/\d+/)?.join(''))).flat().map(Number)
  const isEdgeHorizontalYAxisValueThreshold = 40

  state.isHorizontal = Math.abs(startY - endY) < isEdgeHorizontalYAxisValueThreshold
  setLabelRotation(startX, startY, control2X, control2Y)

  if (params.siblingsAmount === 1) {
    return params.path
  }

  // As position is 0-based, we need to add 1 to it
  const position = params.position + 1
  let offset = position % 2 === 0 ? (position / 2) * 8 : -((position + 1) / 2) * 8

  if (position >= 3) {
    offset = offset * 1.5
  }

  state.edgeOffset = offset

  if (state.isHorizontal) {
    const controlDistanceFromStartOrEnd = Math.abs(control1X - startX) / 2

    const start = `M ${startX},${startY}`
    const control1 = `C ${startX + controlDistanceFromStartOrEnd},${startY + offset}`
    const control2 = `${endX - controlDistanceFromStartOrEnd},${endY + offset}`
    const end = `${endX},${endY}`
    
    const newPath = `${start} ${control1} ${control2} ${end}`
    return newPath
  }

  else {
    const start = `M ${startX},${startY}`
    const control1 = `C ${control1X + offset},${control1Y}`
    const control2 = `${control2X + offset},${control2Y}`
    const end = `${endX},${endY}`

    const newPath = `${start} ${control1} ${control2} ${end}`
    return newPath
  }
}

function setLabelRotation(control1X: number, control1Y: number, control2X: number, control2Y: number) {
  if(siblingsAmount.value === 1) {
    state.labelRotation = 0
    return
  }

  const rotation = Math.atan2(control2Y - control1Y, control2X - control1X) * 180 / Math.PI
  state.labelRotation = rotation > 90 || rotation < -90 ? rotation + 180 : rotation
}

</script>

<template>
  
  <!-- Displayed edge -->
  <path :id="props.id" ref="curve" :style="props.style" class="vue-flow__edge-path" :d="path[0]" :marker-end="props.markerEnd" />

  <!-- Hitbox -->
  <path fill="none" ref="hitbox" class="vue-flow__edge-interaction" :d="path[0]" :stroke-width="siblingsAmount.length >= 2 ? '20' : '10'" stroke-opacity="0" />

  <EdgeLabelRenderer>
    <div 
      :style="{
        pointerEvents: 'none',
        position: 'absolute',
        transform: `translate(-50%, -50%) translate(${path[1]}px,${path[2]}px) translate(${labelOffset}) rotate(${state.labelRotation}deg)`
      }"
      class="px-0.5 text-[#667085] font-size-[8px] line-height-[12px] bg-white"
    >
      {{ label }}
    </div>
  </EdgeLabelRenderer>
</template>
