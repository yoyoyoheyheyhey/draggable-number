<template>
  <div
    class="d-flex align-center"
    @mousedown.prevent.stop="dragStart"
    @touchstart.prevent.stop="dragStart"
  >
    <label
      :id="label"
      :for="label"
      :style="{ cursor: cursorDirection }"
    >
      <slot name="draggable-item" />
      <span v-if="!hideLabel">{{ label }}</span>
    </label>

    <span
      v-if="readOnly"
      :class="textClass"
    >
      {{ value }}
    </span>

    <v-text-field
      v-else
      type="number"
      :max="max"
      :min="min"
      :name="label"
      :step="step"
      :value="value"
      @input="adjustValue($event.target.value)"
    ></v-text-field>
  </div>
</template>

<script>
import { ref, computed } from '@vue/composition-api'

export default {
  props: {
    dragDirection: {
      type: String,
      default: 'X',
      validator: value => ['X', 'Y'].indexOf(value) !== -1,
    },
    value: {
      type: [Number, String],
      validator: value => !Number.isNaN(Number(value)),
      required: true,
    },
    max: {
      type: [Number, String],
      validator: value => !Number.isNaN(Number(value)),
      default: 100,
    },
    min: {
      type: [Number, String],
      validator: value => !Number.isNaN(Number(value)),
      default: -100,
    },
    step: {
      type: [Number, String],
      validator: value => !Number.isNaN(Number(value)),
      default: 1,
    },
    label: {
      type: String,
      default: 'Draggable Number',
    },
    hideLabel: {
      type: Boolean,
      default: false,
    },
    readOnly: {
      type: Boolean,
      default: false,
    },
    textClass: {
      type: Array,
      default: () => [],
    },
  },
  setup(props, { emit }) {
    const isDragging = ref(false)
    const previousTouch = ref()

    const cursorDirection = computed(() => {
      return props.dragDirection === 'Y' ? 'ns-resize' : 'ew-resize'
    })

    const adjustValue = val => {
      const whichEvent = {
        mouse: val instanceof MouseEvent,
        touch: val instanceof TouchEvent,
      }

      let newVal

      if (whichEvent.mouse) {
        newVal = props.dragDirection === 'Y' ? -val.movementY : val.movementX
        newVal = Number(props.value + newVal)
      }

      if (whichEvent.touch) {
        const touch = val.touches[0]

        if (previousTouch.value) {
          const movementX = touch.pageX - previousTouch.value.pageX
          const movementY = touch.pageY - previousTouch.value.pageY
          const movementToCalc = props.dragDirection === 'Y' ? -movementY : movementX

          newVal = Math.ceil(Number(props.value) + (movementToCalc / 1000)) // 割10くらいが感度として丁度いいので
        } else {
          previousTouch.value = touch
          newVal = Number(props.value)
        }
      }

      if (!whichEvent.mouse && !whichEvent.touch) newVal = Number(val)

      if (props.min && newVal < props.min) newVal = Number(props.min)
      if (props.max && newVal > props.max) newVal = Number(props.max)

      emit('input', newVal)
    }

    const dragEnd = () => {
      isDragging.value = false
      document.body.style.cursor = ''
      document.body.style.userSelect = ''

      document.removeEventListener('mousemove', adjustValue)
      document.removeEventListener('mouseup', dragEnd)

      document.removeEventListener('touchmove', adjustValue)
      document.removeEventListener('touchend', dragEnd)
      document.removeEventListener('touchcancel', dragEnd)
      previousTouch.value = undefined
    }

    const dragStart = () => {
      isDragging.value = true
      document.body.style.cursor = props.cursorDirection
      document.body.style.userSelect = 'none'

      document.addEventListener('mousemove', adjustValue)
      document.addEventListener('mouseup', dragEnd)

      document.addEventListener('touchmove', adjustValue)
      document.addEventListener('touchend', dragEnd)
      document.addEventListener('touchcancel', dragEnd)
    }

    return {
      isDragging,
      cursorDirection,
      adjustValue,
      dragStart,
    }
  },
}
</script>
