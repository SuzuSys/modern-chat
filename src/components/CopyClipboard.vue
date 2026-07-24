<template>
  <v-btn
    :color="copied ? 'success' : undefined"
    :disabled="!text"
    :icon="copied ? 'mdi-check' : 'mdi-content-copy'"
    size="small"
    variant="text"
    @click="copyText"
  />
</template>

<script setup lang="ts">
  import { onBeforeUnmount, ref } from 'vue'

  const props = defineProps<{
    text: string
  }>()

  const copied = ref(false)
  let copiedTimer: ReturnType<typeof setTimeout> | undefined

  async function copyText () {
    if (!props.text) {
      return
    }

    try {
      await navigator.clipboard.writeText(props.text)
      copied.value = true
      clearTimeout(copiedTimer)
      copiedTimer = setTimeout(() => {
        copied.value = false
      }, 1600)
    } catch (error) {
      console.error('Copy failed:', error)
    }
  }

  onBeforeUnmount(() => {
    clearTimeout(copiedTimer)
  })
</script>
