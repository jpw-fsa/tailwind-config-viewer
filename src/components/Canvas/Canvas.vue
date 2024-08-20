<template>
  <div class="bg-gray-100 dark:bg-midnight">
    <div
      v-if="!config"
      style="height: calc(100vh - 63px);"
      class="flex items-center justify-center">
        <p class="text-gray-600 text-center font-bold">Loading Config...</p>
    </div>
    <template v-if="config">
      <div class="pt-8 px-3 flex">
        <div class="hidden md:block flex-none h-full overflow-y-auto top-0 sticky max-h-screen pt-2">
          <ToggleSwitch
            name="dark-mode"
            class="mb-3 ml-3"
            :value="darkMode"
            @input="$emit('toggle-dark-mode', $event)"
            label="Dark Mode"
          />
          <div class="ml-3 text-sm text-gray-700 dark:text-gray-500">Tailwind v{{ config.tailwindVersion }}</div>
          <nav class="pt-3 pr-20 pb-12 px-3 h-full">
            <a
              v-for="section in configTransformed"
              :key="section.title"
              :href="`#${section.title}`"
              class="relative flex items-center py-2 hover:text-gray-900 dark-hover:text-gray-200 text-base rounded-sm"
              :class="[activeSection === section ? 'text-gray-900 dark:text-gray-200' : 'text-gray-700 dark:text-gray-500']"
              @click="setActiveSection(section)"
            >
              <div
                class="absolute rounded-full bg-gray-500 dark:bg-gray-600 transition duration-200"
                :class="[activeSection === section ? 'visible opacity-100' : 'invisible opacity-0']"
                :style="{width: '5px', height: '5px', left: '-12px'}"
              />
              {{ section.title }}
            </a>
          </nav>
        </div>
        <div class="md:pl-4">
          <CanvasSection
            v-for="section in configTransformed"
            :key="section.title"
            :title="section.title"
            :id="section.title"
          >
            <component
              ref="sectionRefs"
              :is="section.component"
              :data="section.data"
              :config="config"
            />
          </CanvasSection>
        </div>
      </div>
    </template>
  </div>
</template>

<script setup>
import { defineProps, provide, ref, onMounted, onUnmounted } from 'vue'
import defu from 'defu'
import { useIntersectionObserver } from '@vueuse/core'
import themeComponentMapper from './themeComponentMapper'
import fontTagCreator from './fontTagCreator'
import CanvasSection from './CanvasSection.vue'
import ToggleSwitch from '../ToggleSwitch.vue'
import defaultOptions from '../../defaultOptions'

defineProps({
  darkMode: {
    type: Boolean,
    required: false
  }
})

const prefixClassName = (className) => {
  return config.value.prefix ? `${config.value.prefix}${className}` : className
}

const getConfig = () => {
  return config.value
}

const setActiveSection = (section) => {
  activeSection.value = section
}

provide('prefixClassName', prefixClassName)
provide('getConfig', getConfig)


const activeSection = ref(null)
const config = ref(null)
const configTransformed = ref(null)
const sectionRefs = ref([])

const { stop } = useIntersectionObserver(
  sectionRefs,
  ([{ isIntersecting, target }], observerElement) => {
    const index = sectionRefs.value.map(ref => ref.$el).indexOf(target)
    
    if (isIntersecting) {
      setActiveSection(configTransformed.value[index])
    } else {
      setActiveSection(null)
    }
  },
  {
    threshold: 0.0,
    rootMargin: '-40% 0px -60% 0px'
  }
)

onMounted(async () => {
  const response = await fetch('/api/config.json')
  config.value = await response.json()
  config.value = defu(config.value, defaultOptions)
  configTransformed.value = themeComponentMapper(config.value.theme)
  fontTagCreator(config.value.theme)
})

onUnmounted(() => {
  stop()
})

</script>
