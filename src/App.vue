<template>
  <div id="app">
   <div id="pdf-viewer">
    <div id="paf-wrapper"></div>
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
    }
  },
  mounted() {
    pdfjsLib.getDocument('https://mozilla.github.io/pdf.js/web/compressed.tracemonkey-pldi-09.pdf').promise.then(pdfDoc_ => {
      this.pdfDoc = pdfDoc_;
      let pagesCount = this.pdfDoc.numPages;
      for (let i = 1; i <= pagesCount; i++) {
        this.renderPage(i);
      }
    })
    // canvas 监听画线
    // this.canvas.addEventListener('mousedown', (e) => {
    //   let isDrawing = true;
    //   this.ctx.beginPath();
    //   this.ctx.moveTo(e.offsetX, e.offsetY);
    //   this.canvas.addEventListener('mousemove', (e) => {
    //     if (isDrawing) {
    //       this.ctx.lineTo(e.offsetX, e.offsetY);
    //       this.ctx.stroke();
    //     }
    //   })
    //   this.canvas.addEventListener('mouseup', () => {
    //     isDrawing = false;

    //   })
    //   this.canvas.addEventListener('mouseout', () => {
    //     isDrawing = false;
    //   })
    // })

  },
  methods: {
    renderPage(num) {
      this.pageRendering = true;
      let pafWrapper = document.getElementById('paf-wrapper'); // #paf-wrapper
      let canvasId = 'canvas' + num;
     
      // Using promise to fetch the page
      this.pdfDoc.getPage(num).then(page => {
        const viewport = page.getViewport({ scale: this.scale });
        let viewportWidth =  viewport.height;
        let viewportHeight =viewport.width;
        // Prepare canvas using PDF page dimensions
        let canvas = document.createElement('canvas');
        let ctx = canvas.getContext('2d');
        canvas.id = canvasId;
        canvas.height = viewportWidth;
        canvas.width = viewportHeight;
        // Render PDF page into canvas context
        var renderContext = {
          canvasContext:ctx,
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
        pafWrapper.appendChild(canvas);
        //为每个canvas添加监听事件
        this.addCanvasEvent(canvasId);
      });
    },
    // 根据传递页码添加cavas监听时间
    addCanvasEvent(canvasId) {
      let canvas = document.getElementById(canvasId);
      let ctx = canvas.getContext('2d');
      canvas.addEventListener('mousedown', (e) => {
        let isDrawing = true;
        ctx.beginPath();
        ctx.moveTo(e.offsetX, e.offsetY);
        canvas.addEventListener('mousemove', (e) => {
          if (isDrawing) {
            ctx.lineTo(e.offsetX, e.offsetY);
            ctx.stroke();
          }
        })
        canvas.addEventListener('mouseup', () => {
          isDrawing = false;
        })
        canvas.addEventListener('mouseout', () => {
          isDrawing = false;
        })
      })

    },

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
