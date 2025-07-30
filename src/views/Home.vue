<template>
  <div class="container mx-auto px-4 py-8 max-w-6xl">
    <header class="text-center mb-12">
      <h1 class="text-4xl font-bold text-gray-900 mb-4">Vue Image to Component Converter</h1>
      <p class="text-xl text-gray-600">Upload an image and get a Vue component generated from it</p>
    </header>

    <div class="grid grid-cols-1 lg:grid-cols-2 gap-8">
      <!-- Image Upload Section -->
      <div class="bg-white rounded-lg shadow-lg p-6">
        <h2 class="text-2xl font-semibold mb-4">Upload Image</h2>
        
        <div 
          class="border-2 border-dashed border-gray-300 rounded-lg p-8 text-center hover:border-blue-500 transition-colors cursor-pointer"
          @click="triggerFileInput"
          @dragover.prevent="isDragging = true"
          @dragleave.prevent="isDragging = false"
          @drop.prevent="handleDrop"
          :class="{ 'border-blue-500 bg-blue-50': isDragging }"
        >
          <input 
            ref="fileInput" 
            type="file" 
            accept="image/*" 
            @change="handleFileSelect" 
            class="hidden"
          >
          
          <div v-if="!selectedImage">
            <svg class="mx-auto h-12 w-12 text-gray-400 mb-4" stroke="currentColor" fill="none" viewBox="0 0 48 48">
              <path d="M28 8H12a4 4 0 00-4 4v20m32-12v8m0 0v8a4 4 0 01-4 4H12a4 4 0 01-4-4v-4m32-4l-3.172-3.172a4 4 0 00-5.656 0L28 28M8 32l9.172-9.172a4 4 0 015.656 0L28 28m0 0l4 4m4-24h8m-4-4v8m-12 4h.02" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" />
            </svg>
            <p class="text-gray-600">Click to upload or drag and drop</p>
            <p class="text-sm text-gray-400">PNG, JPG, GIF up to 10MB</p>
          </div>
          
          <div v-else>
            <img :src="selectedImage" alt="Selected" class="max-w-full h-auto rounded-lg">
            <button @click.stop="removeImage" class="mt-4 bg-red-500 text-white px-4 py-2 rounded hover:bg-red-600">
              Remove Image
            </button>
          </div>
        </div>

        <div v-if="selectedImage" class="mt-6">
          <button 
            @click="analyzeImage"
            :disabled="isAnalyzing"
            class="w-full bg-blue-600 text-white py-3 px-4 rounded-lg hover:bg-blue-700 disabled:opacity-50 disabled:cursor-not-allowed"
          >
            {{ isAnalyzing ? 'Analyzing...' : 'Generate Vue Component' }}
          </button>
        </div>
      </div>

      <!-- Generated Component Section -->
      <div class="bg-white rounded-lg shadow-lg p-6">
        <h2 class="text-2xl font-semibold mb-4">Generated Vue Component</h2>
        
        <div v-if="!generatedComponent && !isAnalyzing" class="text-center py-12 text-gray-500">
          Upload an image to generate a Vue component
        </div>

        <div v-if="isAnalyzing" class="text-center py-12">
          <div class="animate-spin rounded-full h-12 w-12 border-b-2 border-blue-600 mx-auto mb-4"></div>
          <p class="text-gray-600">Analyzing image and generating component...</p>
        </div>

        <div v-if="generatedComponent">
          <div class="mb-4 flex justify-between items-center">
            <h3 class="text-lg font-medium">{{ generatedComponent.filename }}</h3>
            <div class="space-x-2">
              <button @click="copyToClipboard" class="bg-green-600 text-white px-3 py-1 rounded text-sm hover:bg-green-700">
                {{ copied ? 'Copied!' : 'Copy' }}
              </button>
              <button @click="downloadComponent" class="bg-blue-600 text-white px-3 py-1 rounded text-sm hover:bg-blue-700">
                Download
              </button>
            </div>
          </div>
          
          <div class="bg-gray-900 rounded-lg p-4 overflow-x-auto">
            <pre class="text-green-400 text-sm"><code>{{ generatedComponent.code }}</code></pre>
          </div>

          <div v-if="generatedComponent.preview" class="mt-6">
            <h3 class="text-lg font-medium mb-2">Preview</h3>
            <div class="border rounded-lg p-4 bg-gray-50">
              <div v-html="generatedComponent.preview"></div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Analysis Results -->
    <div v-if="analysisResults" class="mt-8 bg-white rounded-lg shadow-lg p-6">
      <h2 class="text-2xl font-semibold mb-4">Analysis Results</h2>
      <div class="grid grid-cols-1 md:grid-cols-3 gap-4">
        <div class="bg-blue-50 p-4 rounded-lg">
          <h3 class="font-semibold text-blue-800">Detected Elements</h3>
          <ul class="mt-2 text-sm text-blue-700">
            <li v-for="element in analysisResults.elements" :key="element">• {{ element }}</li>
          </ul>
        </div>
        <div class="bg-green-50 p-4 rounded-lg">
          <h3 class="font-semibold text-green-800">Color Palette</h3>
          <div class="mt-2 flex flex-wrap gap-2">
            <div 
              v-for="color in analysisResults.colors" 
              :key="color"
              class="w-6 h-6 rounded border"
              :style="{ backgroundColor: color }"
              :title="color"
            ></div>
          </div>
        </div>
        <div class="bg-purple-50 p-4 rounded-lg">
          <h3 class="font-semibold text-purple-800">Layout Type</h3>
          <p class="mt-2 text-sm text-purple-700">{{ analysisResults.layout }}</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'

interface AnalysisResults {
  elements: string[]
  colors: string[]
  layout: string
}

interface GeneratedComponent {
  filename: string
  code: string
  preview?: string
}

const fileInput = ref<HTMLInputElement>()
const selectedImage = ref<string>('')
const isDragging = ref(false)
const isAnalyzing = ref(false)
const generatedComponent = ref<GeneratedComponent>()
const analysisResults = ref<AnalysisResults>()
const copied = ref(false)

const triggerFileInput = () => {
  fileInput.value?.click()
}

const handleFileSelect = (event: Event) => {
  const file = (event.target as HTMLInputElement).files?.[0]
  if (file) {
    processFile(file)
  }
}

const handleDrop = (event: DragEvent) => {
  isDragging.value = false
  const file = event.dataTransfer?.files[0]
  if (file) {
    processFile(file)
  }
}

const processFile = (file: File) => {
  const reader = new FileReader()
  reader.onload = (e) => {
    selectedImage.value = e.target?.result as string
  }
  reader.readAsDataURL(file)
}

const removeImage = () => {
  selectedImage.value = ''
  generatedComponent.value = undefined
  analysisResults.value = undefined
  if (fileInput.value) {
    fileInput.value.value = ''
  }
}

const analyzeImage = async () => {
  if (!selectedImage.value) return

  isAnalyzing.value = true
  
  try {
    // Simulate image analysis
    await new Promise(resolve => setTimeout(resolve, 2000))
    
    // Mock analysis results
    analysisResults.value = {
      elements: ['Header', 'Navigation', 'Cards', 'Buttons', 'Footer'],
      colors: ['#3B82F6', '#1F2937', '#F9FAFB', '#10B981', '#EF4444'],
      layout: 'Grid Layout with Cards'
    }

    // Generate Vue component based on analysis
    const componentCode = generateVueComponent(analysisResults.value)
    
    generatedComponent.value = {
      filename: 'GeneratedComponent.vue',
      code: componentCode,
      preview: generatePreview(analysisResults.value)
    }
  } catch (error) {
    console.error('Error analyzing image:', error)
  } finally {
    isAnalyzing.value = false
  }
}

const generateVueComponent = (analysis: AnalysisResults): string => {
  return `<template>
  <div class="generated-component">
    <!-- ${analysis.layout} -->
    <header class="header">
      <nav class="navigation">
        <div class="nav-items">
          <a href="#" class="nav-link">Home</a>
          <a href="#" class="nav-link">About</a>
          <a href="#" class="nav-link">Contact</a>
        </div>
      </nav>
    </header>

    <main class="main-content">
      <div class="grid-container">
        <div v-for="item in items" :key="item.id" class="card">
          <div class="card-header">
            <h3>{{ item.title }}</h3>
          </div>
          <div class="card-content">
            <p>{{ item.description }}</p>
          </div>
          <div class="card-actions">
            <button class="btn btn-primary">Learn More</button>
          </div>
        </div>
      </div>
    </main>

    <footer class="footer">
      <p>&copy; 2024 Generated Component</p>
    </footer>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'

interface Item {
  id: number
  title: string
  description: string
}

const items = ref<Item[]>([
  { id: 1, title: 'Feature One', description: 'Description for feature one' },
  { id: 2, title: 'Feature Two', description: 'Description for feature two' },
  { id: 3, title: 'Feature Three', description: 'Description for feature three' }
])
</script>

<style scoped>
.generated-component {
  min-height: 100vh;
  background: linear-gradient(135deg, ${analysis.colors[0]}, ${analysis.colors[1]});
}

.header {
  background: ${analysis.colors[1]};
  padding: 1rem 0;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.navigation {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 1rem;
}

.nav-items {
  display: flex;
  gap: 2rem;
}

.nav-link {
  color: ${analysis.colors[2]};
  text-decoration: none;
  font-weight: 500;
  transition: color 0.3s ease;
}

.nav-link:hover {
  color: ${analysis.colors[3]};
}

.main-content {
  max-width: 1200px;
  margin: 0 auto;
  padding: 2rem 1rem;
}

.grid-container {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 2rem;
  margin-top: 2rem;
}

.card {
  background: ${analysis.colors[2]};
  border-radius: 8px;
  padding: 1.5rem;
  box-shadow: 0 4px 6px rgba(0,0,0,0.1);
  transition: transform 0.3s ease;
}

.card:hover {
  transform: translateY(-4px);
}

.card-header h3 {
  margin: 0 0 1rem 0;
  color: ${analysis.colors[1]};
}

.card-content p {
  color: #666;
  line-height: 1.6;
  margin-bottom: 1.5rem;
}

.btn {
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: 500;
  transition: background-color 0.3s ease;
}

.btn-primary {
  background: ${analysis.colors[3]};
  color: white;
}

.btn-primary:hover {
  background: ${analysis.colors[0]};
}

.footer {
  background: ${analysis.colors[1]};
  color: ${analysis.colors[2]};
  text-align: center;
  padding: 1rem 0;
  margin-top: 2rem;
}
</style>`
}

const generatePreview = (analysis: AnalysisResults): string => {
  return `
    <div style="background: linear-gradient(135deg, ${analysis.colors[0]}, ${analysis.colors[1]}); padding: 20px; border-radius: 8px;">
      <div style="background: ${analysis.colors[2]}; padding: 15px; border-radius: 6px; margin-bottom: 15px;">
        <h3 style="margin: 0; color: ${analysis.colors[1]};">Generated Preview</h3>
      </div>
      <div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(150px, 1fr)); gap: 10px;">
        ${analysis.elements.slice(0, 3).map((_, i) => `
          <div style="background: ${analysis.colors[2]}; padding: 10px; border-radius: 4px; text-align: center;">
            <div style="background: ${analysis.colors[i + 2] || analysis.colors[0]}; height: 20px; border-radius: 2px; margin-bottom: 8px;"></div>
            <small style="color: #666;">Component ${i + 1}</small>
          </div>
        `).join('')}
      </div>
    </div>
  `
}

const copyToClipboard = async () => {
  if (!generatedComponent.value) return
  
  try {
    await navigator.clipboard.writeText(generatedComponent.value.code)
    copied.value = true
    setTimeout(() => {
      copied.value = false
    }, 2000)
  } catch (error) {
    console.error('Failed to copy to clipboard:', error)
  }
}

const downloadComponent = () => {
  if (!generatedComponent.value) return
  
  const blob = new Blob([generatedComponent.value.code], { type: 'text/plain' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = generatedComponent.value.filename
  document.body.appendChild(a)
  a.click()
  document.body.removeChild(a)
  URL.revokeObjectURL(url)
}
</script>