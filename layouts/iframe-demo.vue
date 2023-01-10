<script setup lang="ts">
import { computed } from '@vue/reactivity'
const props = defineProps<{
  url: string
  scale?: number
}>()
const scaleInvertPercent = computed(() => `${(1 / (props.scale || 1)) * 100}%`)
</script>

<template>
    <div class="bg-section slidev-layout iframe-demo flex flex-col h-full w-full">
        <slot/>
        <div class="h-full w-full">
            <div relative :style="{ width: scaleInvertPercent, height: scaleInvertPercent }">
                <iframe
                id="frame" class="w-full h-full"
                :src="url"
                :style="scale ? { transform: `scale(${scale})`, transformOrigin: 'top left' } : {}"
                />
            </div>
        </div>
    </div>
</template>