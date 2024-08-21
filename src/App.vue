<template>
  <div id="app">
   <div id="pdf-viewer">
    <canvas ref ="pdfCanvas"></canvas>
   </div>
  </div>
</template>

<script>
import * as pdfjsLib from 'pdfjs-dist/legacy/build/pdf'
import PDFJSWorker from 'pdfjs-dist/legacy/build/pdf.worker.entry'

pdfjsLib.GlobalWorkerOptions.workerSrc = PDFJSWorker
export default {
  name: 'App',
  components: {
  },
  data() {
    return {
      pdfDoc: null,
      pageNum: 1,
      pageRendering: false,
      pageNumPending: null,
      scale: 1,
      canvas: null,
      ctx: null
    }
  },
  mounted() {
    this.canvas = this.$refs.pdfCanvas;
    this.ctx = this.canvas.getContext('2d');
    pdfjsLib.getDocument('https://mozilla.github.io/pdf.js/web/compressed.tracemonkey-pldi-09.pdf').promise.then(pdfDoc_ => {
      this.pdfDoc = pdfDoc_;
      this.renderPage(this.pageNum);
    })
  },
  methods: {
    renderPage(num) {
      this.pageRendering = true;
      // Using promise to fetch the page
      this.pdfDoc.getPage(num).then(page => {
        var viewport = page.getViewport({ scale: this.scale });
        // Prepare canvas using PDF page dimensions
        this.canvas.height = viewport.height;
        this.canvas.width = viewport.width;
        // Render PDF page into canvas context
        var renderContext = {
          canvasContext: this.ctx,
          viewport: viewport
        };
        var renderTask = page.render(renderContext);
        // Wait for rendering to finish
        renderTask.promise.then(() => {
          this.pageRendering = false;
          if (this.pageNumPending !== null) {
            // New page rendering is pending
            this.renderPage(this.pageNumPending);
            this.pageNumPending = null;
          }
        });
      });
    }
  }
  
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
