<template>
    <div>
        <div>
            <button id="prev" type="button" @click="onPrevPage">Previous</button>
            <button id="next" type="button" @click="onNextPage">Next</button>
        </div>
        <div>Page: {{ pageNum }} of {{ pageCount }}</div>
        <div id="pdf-container">
            <canvas id="the-canvas" style="direction: ltr"></canvas>
            <div id="text-layer" class="textLayer"></div>
        </div>
    </div>
</template>

<script setup>
import { onMounted, ref } from 'vue'
import * as pdfjsLib from 'pdfjs-dist'
import * as pdfjsWorker from 'pdfjs-dist/build/pdf.worker?url'
import 'pdfjs-dist/web/pdf_viewer.css'

pdfjsLib.GlobalWorkerOptions.workerSrc = pdfjsWorker.default

const url =
    // 'https://un-encrypt-resources.oss-cn-shenzhen.aliyuncs.com/met_new/course/english/reading/20449/20449.pdf'
    // 'https://un-encrypt-resources.oss-cn-shenzhen.aliyuncs.com/met_new/course/english/reading/20425/20425.pdf'
    'https://un-encrypt-resources.oss-cn-shenzhen.aliyuncs.com/met_new/course/english/reading/20435/20435.pdf'

let pdfDoc = null

const pageCount = ref(0)
const pageNum = ref(1)
const pageRendering = ref(null)
const pageNumPending = ref(null)
const scale = ref(3)

let canvas = null
let ctx = null

async function renderPage(num) {
    pageRendering.value = true

    // Using promise to fetch the page
    const page = await pdfDoc.getPage(num)
    const viewport = page.getViewport({ scale: scale.value })
    // Support HiDPI-screens
    const outputScale = window.devicePixelRatio || 1

    // canvas.height = Math.floor(viewport.height * outputScale)
    // canvas.width = Math.floor(viewport.width * outputScale)
    canvas.height = Math.floor(viewport.height * outputScale)
    canvas.width = Math.floor(viewport.width * outputScale)
    // ctx.scale(outputScale, outputScale);
    canvas.style.height = Math.floor(viewport.height) + 'px'
    canvas.style.width = Math.floor(viewport.width) + 'px'
    // canvas.style.height = Math.floor(567 * viewport.height / viewport.width) + 'px'
    // canvas.style.width = Math.floor(567) + 'px'

    // 同步容器大小
    const pdfContainer = document.getElementById('pdf-container')
    pdfContainer.style.height = Math.floor(567 * viewport.height / viewport.width) + 'px'
    pdfContainer.style.width = Math.floor(567) + 'px'

    console.log(outputScale)

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
            renderPage(pageNumPending.value)
            pageNumPending.value = null
        }
        return page.getTextContent()
    }).then((textContent) => {
        console.log('textContent', textContent)
        // Render text layer
        const textLayerDiv = document.getElementById('text-layer')
        textLayerDiv.style.height = Math.floor(viewport.height) + 'px'
        textLayerDiv.style.width = Math.floor(viewport.width) + 'px'
        // textLayerDiv.style.height = Math.floor(567 * viewport.height / viewport.width) + 'px'
        // textLayerDiv.style.width = Math.floor(567) + 'px'
        textLayerDiv.style.setProperty('--scale-factor', scale.value);

        pdfjsLib.renderTextLayer({
            textContentSource: textContent,
            container: textLayerDiv,
            viewport: viewport,
            textDivs: [],
            enhanceTextSelection: true
        })
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

<style scoped>
#pdf-container {
    position: relative;
    width: 100%;
    height: 100%;
}
canvas {
    display: block;
}
.textLayer {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    /* pointer-events: none; */
}
</style>
