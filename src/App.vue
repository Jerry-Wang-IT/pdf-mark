<template>
  <div id="app">
    <div id="container">
      <div id="pdf-viewer">
          <div id="paf-wrapper"></div>
      </div>
      <div v-if="comments.length" class="comments-wrapper">
        <div class="commentItem" v-for="comment in comments" :key="comment.id" @click="scrollToAnnotation(comment.pageNum)">
          <span class="commentTitle">{{`${comment.canvasId}-标注`}}</span>
          <span class="commentText">{{comment.commentText}}</span>
        </div>
      </div>
      <div v-if="isShowInput" class="input-wrapper">
        <input type="text" v-model="commentText">
        <button @click="submitComment">提交</button>
      </div>
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
      pageHeight: 0,
      pageRendering: false,
      pageNumPending: null,
      scale: 1,
      annotations: [],
      isDrawing: false,
      comments:[],
      isShowInput: false,
      currentPage:1,
      currentAnnotationId:'',
      commentText:'',
    }
  },
  mounted() {
    // 
    pdfjsLib.getDocument('https://mozilla.github.io/pdf.js/web/compressed.tracemonkey-pldi-09.pdf').promise.then(pdfDoc_ => {
      this.pdfDoc = pdfDoc_;
      let pagesCount = this.pdfDoc.numPages;
      for (let i = 1; i <= pagesCount; i++) {
        this.renderPage(i);
      }
    })
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
        this.pageHeight = viewportHeight;
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
        this.addCanvasEvent(canvasId, num);
       // #paf-wrapper 
        //为每个canvas添加监听事件
      });
    },
    // 根据传递页码添加cavas监听时间
    addCanvasEvent(canvasId, num) {
      let self = this;
      let canvas = document.getElementById(canvasId);
      let ctx = canvas.getContext('2d');
      let startX = 0;
      let startY = 0;
      let points = [];
      let pageNum = num;
      canvas.addEventListener('mousedown', (e) => {
        self.isDrawing = true;
        startX = e.offsetX;
        startY = e.offsetY;
        ctx.strokeStyle = 'red';
        ctx.lineWidth = 2;
        ctx.beginPath();
        ctx.moveTo(startX, startY);
        points.push({ x: startX, y: startY });
        canvas.addEventListener('mousemove', (e) => {
          if (self.isDrawing) {
            let currentX = e.offsetX;
            let currentY = e.offsetY;
            ctx.lineTo(currentX, currentY);
            ctx.stroke();
            points.push({ x: currentX, y: currentY });
          }
        })
        canvas.addEventListener('mouseup', () => {
          self.isDrawing = false;
          self.isShowInput = true;
          self.currentPage = pageNum;
          //保存数据和关联评论
          self.saveAnnotationData(canvasId,points,pageNum);
        })
      })
    },
    saveAnnotationData(canvasId,points,pageNum,startY){
      let self = this
      let id = self.randomId();
      let annotation ={
        canvasId,
        id,
        shapStyle:{
          strokeStyle:'red',
          lineWidth:2
        },
        type:'line',
        points,
        pageNum,
        height:startY
      };
      self.annotations= self.annotations.concat(annotation);
      self.isShowInput = true;
      self.currentPage = pageNum;
      self.currentAnnotationId = id;
    },
    //提交评论
    submitComment(){
      let commentText = this.commentText;
      if(!commentText){
        alert('请输入评论内容')
        return;
      }else{
        //保存评论
        let annotation = this.annotations.find(item => item.id === this.currentAnnotationId);
        let {pageNum,canvasId} = annotation;
        let comment = {
          id: this.currentAnnotationId,
          pageNum,
          canvasId,
          commentText
        }
        this.comments.push(comment);
        this.commentText = '';
        this.isShowInput = false;
        //关联标注并保存数据到服务器
        this.updateCommentDataAndSaveAnnotations(comment);
      }
      
    },
    //关联标注并保存数据到服务器
    updateCommentDataAndSaveAnnotations(comment){
      this.annotations.map(item => {
        if(item.id === comment.id){
          item.comment = comment;
        }
      });
      console.log(this.annotations)
    },
    //生成一个10位数字母和数字的随机数
    randomId() {
      return Math.random().toString(36).substr(2, 10);
    },

    // 点击评论滚动到对应页面
    scrollToAnnotation(pageNum) {
      let pdfViewer = document.getElementById('pdf-viewer');
      let pageHeight = this.pageHeight + 100;
      let scrollTop = --pageNum * pageHeight;
      pdfViewer.scrollTo(0, scrollTop);
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
  height: 100vh;
  width: 100vw;
  box-sizing: border-box;
}
#container{
  width: 100%;
  height: 100%;
  position: relative;

}
#pdf-viewer{
  width: 100%;
  height: 100%;
  box-sizing: border-box;
  display: flex;
  flex-direction: row;
  justify-content: center;
  overflow-y: auto;
}
#paf-wrapper{
  display: flex;
  flex-direction: column;
}
.comments-wrapper{
  width: 300px;
  height: 600px;
  border: 1px solid #CCC;
  position: fixed;
  right: 0;
  top: 60px;
  border-radius: 12px;
  padding:12px 30px;
  overflow-y: scroll;
}
.input-wrapper{
  position: fixed;
  width: 300px;
  height:40px;
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  right: 0;
  bottom: 30px;
  border-radius: 12px;
  padding:12px 30px;
}
.input-wrapper input{
  display: inline-block;
  width: 200px;
  height: 40px;
  border: 1px solid #CCC;
  border-radius: 12px;
  padding: 0 12px;
}
.input-wrapper button{
  display: inline-block;
  width: 60px;
  height: 40px;
  border: 1px solid #CCC;
  border-radius: 12px;
  padding: 0 12px;
}
.commentItem{
  text-align: left;
  height:25px;
  background-color: #999;
  line-height: 25px;
  margin-top: 5px;
}
.commentText{
  margin-left: 10px;
  color:#ff5651;
}

</style>
