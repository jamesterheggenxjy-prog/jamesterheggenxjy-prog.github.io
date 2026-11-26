<script lang="ts">
import { defineComponent, onMounted, watch, onUnmounted } from 'vue'
import { useStore } from '../store'
import { BAClickFX } from 'ba-click-fx'

export default defineComponent({
  setup() {
    const { state } = useStore()
    let fxInstance: BAClickFX | null = null

    function createFX() {
      fxInstance?.destroy()
      const color: [number, number, number] = state.darkMode === 'dark'
        ? [252, 146, 174]
        : [105, 161, 255]

      fxInstance = new BAClickFX({
        color,
        scale: 1.0,
        opacity: 0.5,
        trailEnabled: true,
        trailAlways: false,
      })
    }

    onMounted(createFX)
    watch(() => state.darkMode, createFX)
    onUnmounted(() => fxInstance?.destroy())

    // The effect owns its canvas. Vue renders the same placeholder on both sides.
    return () => null
  },
})
</script>
