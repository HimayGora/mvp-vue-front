<script setup>
import { ref, computed, onMounted } from 'vue'
import axios from 'axios'
import { API_ENDPOINTS } from '../utils/api'
import { useRouter } from 'vue-router';
import { useHead } from '@vueuse/head';

const features = ref(localStorage.getItem('features') || '')
const isLoading = ref(false)
const generatedText = ref('')
const errorMessage = ref('')
const router = useRouter();

const MAX_WORDS = 100
const wordCount = ref(0)
const remainingWords = computed(() => MAX_WORDS - wordCount.value)

const handleInput = () => {
  const words = features.value.trim().match(/\b\w+([-']\w+)*\b/g) || []
  wordCount.value = words.length
  if (wordCount.value > MAX_WORDS) {
    features.value = words.slice(0, MAX_WORDS).join(' ')
    wordCount.value = MAX_WORDS
  }
  localStorage.setItem('features', features.value)
  errorMessage.value = ''
}

const generateContent = async () => {
  isLoading.value = true
  errorMessage.value = ''
  generatedText.value = ''

  try {
    const response = await axios.post(API_ENDPOINTS.generate(), {
      features: features.value.trim()
    })
    generatedText.value = response.data.generatedText
  } catch (error) {
    console.error('Error generating content:', error)
    if (error.response?.status === 401) {
      errorMessage.value = 'Your session has expired. Please log in again.'
      router.push('/login')
    } else {
      errorMessage.value = error.response?.data?.error || 'An error occurred while generating content.'
    }
  } finally {
    isLoading.value = false
  }
}

const handleLogout = async () => {
  try {
    await axios.post(API_ENDPOINTS.logout())
    router.push('/login')
  } catch (error) {
    console.error('Error logging out:', error)
    errorMessage.value = 'Failed to log out. Please try again.'
  }
}

const copyGeneratedText = async () => {
  try {
    await navigator.clipboard.writeText(generatedText.value)
  } catch {
    errorMessage.value = 'Unable to copy. Please select manually.'
  }
}

const resetForm = () => {
  features.value = ''
  wordCount.value = 0
  generatedText.value = ''
  localStorage.removeItem('features')
}

onMounted(() => handleInput())

useHead({
  title: 'Generator | BenefitGen'
})
</script>

<template>
  <div class="w-full max-w-2xl">
    <div class="flex justify-between items-center mb-6">
      <h1 class="text-3xl font-bold text-amber-400">Benefit Copy Generator</h1>
      <button @click="handleLogout" class="bg-gray-600 hover:bg-gray-700 text-white font-bold py-2 px-4 rounded-lg text-sm transition-colors duration-300">
        Logout
      </button>
    </div>

    <form @submit.prevent="generateContent" class="bg-gray-800 shadow-md rounded-lg px-8 pt-6 pb-8 mb-4">
      <div class="mb-6">
        <label for="features" class="block text-gray-300 text-sm font-bold mb-2">What does your product do?</label>
        <p class="text-gray-400 text-sm mb-2">
          Paste a short feature list or a 1–2 sentence description. We’ll turn it into benefit copy.
        </p>
        <textarea
          v-model="features"
          @input="handleInput"
          :disabled="isLoading"
          id="features"
          rows="6"
          placeholder="e.g., Tracks employee time spent on projects vs replying to emails..."
          class="shadow border rounded w-full py-2 px-3 bg-gray-700 text-gray-200 border-gray-600 leading-tight focus:outline-none focus:shadow-outline focus:border-amber-500"
        ></textarea>
        <p class="text-sm mt-1 text-right text-gray-400">
          {{ wordCount }} / {{ MAX_WORDS }} words used
        </p>
        <p v-if="errorMessage" class="text-red-500 text-xs italic mt-2">{{ errorMessage }}</p>
      </div>

      <div class="flex items-center justify-between">
        <button
          type="submit"
          :disabled="isLoading || wordCount === 0"
          class="bg-amber-500 hover:bg-amber-600 text-white font-bold py-3 px-8 rounded-lg focus:outline-none focus:shadow-outline transition-colors duration-300 disabled:bg-gray-500 disabled:cursor-not-allowed"
        >
          {{ isLoading ? 'Generating...' : 'Generate Benefits' }}
        </button>

        <button
          type="button"
          @click="resetForm"
          class="text-sm text-gray-400 hover:underline ml-4"
          v-if="features || generatedText"
        >
          Clear & Start Over
        </button>
      </div>
    </form>

    <div v-if="generatedText" class="bg-gray-800 shadow-md rounded-lg p-6 mt-6">
      <h2 class="text-xl font-bold mb-4 text-amber-400">Generated Result:</h2>
      <p class="text-gray-300 whitespace-pre-wrap">{{ generatedText }}</p>
      <button
        @click="copyGeneratedText"
        class="mt-4 text-sm text-amber-400 hover:underline"
      >
        Copy to Clipboard
      </button>
    </div>
  </div>
</template>

<style scoped>
textarea:disabled {
  background-color: #4b5563;
  opacity: 0.6;
  cursor: not-allowed;
}
</style>