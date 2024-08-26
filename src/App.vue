<template>
    <div>
        <div>
            <canvas id="the-canvas" style="border: 1px solid orange; direction: ltr"></canvas>
        </div>
        <div>
            <button id="prev" type="button" @click="onPrevPage">Previous</button>
            <button id="next" type="button" @click="onNextPage">Next</button>
        </div>
        <div>Page: {{ pageNum }} of {{ pageCount }}</div>
    </div>
</template>

<script setup>
import { onMounted, ref } from 'vue'
import * as pdfjsLib from 'pdfjs-dist'
import * as pdfjsWorker from 'pdfjs-dist/build/pdf.worker?url'
import 'pdfjs-dist/web/pdf_viewer.css'

pdfjsLib.GlobalWorkerOptions.workerSrc = pdfjsWorker.default

const url =
    'https://un-encrypt-resources.oss-cn-shenzhen.aliyuncs.com/met_new/course/english/reading/20449/20449.pdf?'

let pdfDoc = null

const pageCount = ref(0)
const pageNum = ref(1)
const pageRendering = ref(null)
const pageNumPending = ref(null)
const scale = ref(1)

let canvas = null
let ctx = null

async function renderPage(num) {
    pageRendering.value = true

    // Using promise to fetch the page
    const page = await pdfDoc.getPage(num)
    const viewport = page.getViewport({ scale: scale.value })
    // Support HiDPI-screens
    const outputScale = window.devicePixelRatio || 1

    canvas.height = Math.floor(viewport.height * outputScale)
    canvas.width = Math.floor(viewport.width * outputScale)
    canvas.style.height = Math.floor(viewport.height) + 'px'
    canvas.style.width = Math.floor(viewport.width) + 'px'

    const transform = outputScale !== 1 ? [outputScale, 0, 0, outputScale, 0, 0] : null

    // Render PDF page into canvas context
    const renderContext = {
        canvasContext: ctx,
        transform,
        viewport
    }
    const renderTask = page.render(renderContext)

    // Wait for rendering to finish
    renderTask.promise.then(() => {
        pageRendering.value = false
        if (pageNumPending.value !== null) {
            // New page rendering is pending
            renderPage(pageNumPending)
            pageNumPending.value = null
        }
    })
}

/**
 * If another page rendering in progress, waits until the rendering is
 * finised. Otherwise, executes rendering immediately.
 * @param {*} num
 */
function queueRenderPage(num) {
    if (pageRendering.value) {
        pageNumPending.value = num
    } else {
        renderPage(num)
    }
}

function onPrevPage() {
    if (pageNum.value <= 1) {
        return
    }
    pageNum.value--
    queueRenderPage(pageNum.value)
}

function onNextPage() {
    if (pageNum.value >= pageCount.value) {
        return
    }
    pageNum.value++
    queueRenderPage(pageNum.value)
}

onMounted(async () => {
    canvas = document.getElementById('the-canvas')
    ctx = canvas.getContext('2d')
    // Asynchronous download PDF
    const loadingTask = pdfjsLib.getDocument(url)
    pdfDoc = await loadingTask.promise
    pageCount.value = pdfDoc.numPages

    renderPage(pageNum.value)
})
</script>

<style scoped></style>
