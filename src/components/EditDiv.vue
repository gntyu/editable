<template>
  <div class="input-msg"
       id="input-msg"
       ref="msg"
       contenteditable="true"
       @input="changeText"
       @click="handleFocus"
       @keyup="handleKeyup">
  </div>
</template>
 
<script>
export default {
  data(){
    return{
      sel:null,
      range:null,
    }
  },
  name: 'edit',
  props: {
    value: {
      type: String,
      default: ''
    }
  },
  watch: {
    value(newvalue,oldvalue) {
      this.sel=null;
      this.range=null;
      if (!newvalue) {
        // 清空节点内所有html来清空文本
        this.$refs.msg.innerHTML = '';
      }else{
        this.getAtSel();
      }
    }
  },
  methods: {
    getAtSel(){
      var { sel, range } = this.getSel();
      if(!sel||!range)return;
      range.deleteContents();
      const clone = range.cloneRange();
      if(clone.startOffset>=1){
        clone.setStart(clone.commonAncestorContainer,clone.startOffset-1);
        if(clone.toString()=='@'){//新输入的@ 响应@后续操作
          this.sel=sel;
          this.range=range;
        }
      }

    },
    addAt(text){
      let { sel , range } = this;
      if(!sel||!range)return;
      const parent =range.commonAncestorContainer;
      if(range.startOffset-1>=0){
        range.setStart(parent,range.startOffset-1);//去掉手写的@
        range.deleteContents();
      }
      var el = document.createElement("div");
      const random ='at_'+ Math.round(Math.random()*100000);
      el.innerHTML = `<span class="at-inserted" data-name="@${text}" id="${random}">@${text}</span> `;
      var frag = document.createDocumentFragment(), node, lastNode;
      while ( (node = el.firstChild) ) {
        // console.log('el.firstChild',el.firstChild)
        lastNode = frag.appendChild(node);
      }
      range.insertNode(frag);

      if (lastNode) {//网上的
        range = range.cloneRange();
        range.setStartAfter(lastNode);
        range.collapse(true);
        sel.removeAllRanges();
        sel.addRange(range);
      }
      // this.setCaretPositionEnd(random)//自己写的

      this.$emit('input', this.$el.innerHTML);//触发watch更新
      
    },
    changeText() {
      this.$emit('input', this.$el.innerHTML);
      this.checkAt();
    },
    handleKeyup(e){
      if((e.key=='ArrowLeft'||e.key=='ArrowRight')){//左右移动光标
        this.handleFocus(e.key,e.key=='ArrowLeft')
      }
      
    },
    handleFocus(e,front=true){//focus到at-span时,自动移到前面，防止编辑
      const { sel,range }=this.getSel();
      const node = range.commonAncestorContainer.parentNode;
      console.log(range,node)
      if(node&&node.className=="at-inserted"){
        var _range = range.cloneRange();
        if(front){
          _range.setStartBefore(node);
        }else{
          if(node.nextSibling&&!node.nextSibling?.data){
            node.nextSibling.data=" "
          }else if(!node.nextSibling){
            document.querySelector("#input-msg").appendChild(document.createTextNode(" "))
          }
          _range.setStart(node.nextSibling,1);
        }
        _range.collapse(true);
        sel.removeAllRanges();
        sel.addRange(_range);
      }
    },
    checkAt(){
      const atList =document.querySelectorAll("#input-msg .at-inserted")||{};
      const fontList =document.querySelectorAll("#input-msg font")||{};
      atList.forEach(el=>{
        const noData=el.nextSibling&&!el.nextSibling?.data;
        if(el.innerText!==el.dataset.name){
          el.parentNode.removeChild(el);
        }else if(noData||!el.nextSibling){
          const { sel,range }=this.getSel();
          if(noData){
            el.nextSibling.data=" "
          }
          if(!el.nextSibling){
            document.querySelector("#input-msg").appendChild(document.createTextNode(" "))
          }
          range.setStart(el.nextSibling,1);
          range.collapse(true);
          sel.removeAllRanges();
          sel.addRange(range);
        }
      })

      fontList.forEach(el=>{
        el.innerHTML=el.innerText;
      })
    },

    setCaretPositionEnd(random) { 
      var textDom=document.querySelector(`#${random}`);
      textDom.focus();//textDom.nextSibling不能focus
      var range = document.createRange();
      range.setStart(textDom.nextSibling,1);//focus到后面兄弟节点
      range.setEnd(textDom.nextSibling,1);
      var {sel} = this.getSel();
      sel.removeAllRanges();
      sel.addRange(range);
      sel.collapseToStart();//折叠并移至sel的最前，如果可编辑就会闪烁
    },
    getSel(){
      var sel,range;
      if (window.getSelection) {// IE9 and non-IE
        sel = window.getSelection();
        if (sel.getRangeAt && sel.rangeCount) {
          range = sel.getRangeAt(0);
        }
      }else if (document.selection) {//ie10 9 8 7 6 5
        range = document.selection.createRange();//创建选择对象
      }
      return {sel,range}
    },
    enterMsg() {
      this.$emit('send');
    },
    keepLastIndex(obj) {
      var range
      if (window.getSelection) {
        //ie11 10 9 ff safari
        obj.focus(); //解决ff不获取焦点无法定位问题
        range = window.getSelection(); //创建range
        range.selectAllChildren(obj); //range 选择obj下所有子内容
        range.collapseToEnd(); //光标移至最后
      } else if (document.selection) {
        //ie10 9 8 7 6 5
        range = document.selection.createRange(); //创建选择对象
        //var range = document.body.createTextRange();
        range.moveToElementText(obj); //range定位到obj
        range.collapse(false); //光标移至最后
        range.select();
      }
    }
  }
};
</script>
 
<style >
.input-msg {
  line-height: 17px;
  min-height: 17px;
  border: 0;
  display: block;
  flex: 1;
  width: 0;
  max-height: 57px;
  overflow-y: scroll;
  overflow-x: hidden;
  background-color: #f9f8fa;
  padding: 2px 2px 0 2px;
  user-select: text;
  -webkit-user-modify: read-write-plaintext-only;
  
}

.at-inserted{
  color: #1c4cba;
  padding: 0 2px;
}

.input-msg:focus {
  border: 0;
  outline: 0;
}
 
.input-msg:empty:before {
  content: '';
}


</style>