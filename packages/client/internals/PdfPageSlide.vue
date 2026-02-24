<script setup lang="ts">
import * as pdfjsLib from 'pdfjs-dist'
import workerUrl from 'pdfjs-dist/build/pdf.worker.min.mjs?url'
import { onMounted, onUnmounted, ref, watch } from 'vue'
import { useSlideContext } from '../context'

pdfjsLib.GlobalWorkerOptions.workerSrc = workerUrl

const { $page } = useSlideContext()

const canvasRef = ref<HTMLCanvasElement | null>(null)

let pdfDocPromise: Promise<pdfjsLib.PDFDocumentProxy> | null = null
let currentRenderTask: pdfjsLib.RenderTask | null = null

function getPdfDocument() {
  if (!pdfDocPromise)
    pdfDocPromise = pdfjsLib.getDocument('/__slidev/pdf').promise
  return pdfDocPromise
}

async function renderPage(pageNumber: number) {
  if (!canvasRef.value)
    return

  if (currentRenderTask) {
    currentRenderTask.cancel()
    currentRenderTask = null
  }

  try {
    const pdfDoc = await getPdfDocument()
    const page = await pdfDoc.getPage(pageNumber)
    const canvas = canvasRef.value
    if (!canvas)
      return

    const ctx = canvas.getContext('2d')
    if (!ctx)
      return

    const viewport = page.getViewport({ scale: 2 })
    canvas.width = viewport.width
    canvas.height = viewport.height

    currentRenderTask = page.render({ canvasContext: ctx, viewport })
    await currentRenderTask.promise
    currentRenderTask = null
  }
  catch (e: any) {
    if (e?.name !== 'RenderingCancelledException')
      console.error('[PdfPageSlide] failed to render page', pageNumber, e)
  }
}

watch($page, (page) => {
  renderPage(page)
})

onMounted(() => {
  renderPage($page.value)
})

onUnmounted(() => {
  if (currentRenderTask) {
    currentRenderTask.cancel()
    currentRenderTask = null
  }
})
</script>

<template>
  <canvas
    ref="canvasRef"
    style="position: absolute; inset: 0; width: 100%; height: 100%;"
  />
</template>
