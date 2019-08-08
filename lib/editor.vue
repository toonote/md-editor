<template>
<section class="editor" v-on:keydown="onKeydown($event)">
	<div
		id="ace_container"
		v-on:dragover.prevent="onDragOver"
		v-on:drop.prevent.stop="onDrop"
		v-on:paste="onPaste"
	></div>
	<div class="find" :class="{show:isSearching}">
		<form @submit.prevent.stop="onSearch">
			<button type="submit">查找</button>
			<div class="findInputWrapper">
				<input type="text" ref="findInput" v-model="searchKeyword" />
			</div>
		</form>
	</div>
</section>
</template>


<script>
import {throttle} from 'lodash';
import ace from 'brace';
import 'brace/theme/tomorrow';
import 'brace/mode/markdown';
import shortcut from './shortcut';

let _aceEditor;
let _id, _value;

let getExt = (filename) => {
	var ext = '';
	var parts = filename.split('.');
	if(parts.length === 1){
		return ext;
	}
	return '.' + parts[parts.length - 1];
};

export default {
	props: ['value'],
	methods:{
		onDragOver(){
			// console.log('dragover');
		},
		onDrop(e){
			let attachment = e.dataTransfer.files[0];
			if(!attachment/*  || !/^image/.test(attachment.type) */) return;
			let ext = getExt(attachment.name);
			this.$emit('save-attachment', attachment.path, ext);
			// let imagePath = io.saveImage(img.path, ext);
			// this.insertImg(imagePath);
			// logger.ga('send', 'event', 'editor', 'insertImg', 'drag');
		},
		onPaste(e){
			if(!e.clipboardData.items || !e.clipboardData.items.length) return;
			// 判断是否有图片
			let hasImage = false;
			for(let i = e.clipboardData.items.length;i--;){
				let item = e.clipboardData.items[i];
				if(/^image/.test(item.type)){
					hasImage = true;
				}
			}

			// 判断是否是表格
			// 例如从numbers复制出来的会同时有image/png和text
			let isTable = true;
			let text = e.clipboardData.getData('text/plain');
			// 如果没有\n，认为不是表格
		    if(!/\n/.test(text)){
		    	isTable = false;
		    }
			let rows = text.split('\n').filter(row => row);
		    // 如果有不含\t的，认为不是表格
		    if(rows.some(row => !/\t/.test(row))){
		        isTable = false;
		    }

			// 插入图片
			if(hasImage && !isTable){
				this.$emit('save-attachment', '@clipboard');
			}else if(isTable){
				this.insertTable(rows);
			}

		},
		insertAttachment(data){
			let url = data.url;
			_aceEditor.insert(`\n\n![${data.filename}](${url})\n\n`);
			/* if(imagePath){
				imagePath = encodeURI(imagePath);
				if(process.platform === 'win32'){
					// 把\替换回来
					imagePath = imagePath.replace(/%5C/g,'\\');
				}
				_aceEditor.insert(`\n\n![${name}](${imagePath})\n\n`);
			}else{
				_aceEditor.insert(`插入图片出错！`);
			} */
			this.onEditorInput();
		},
		insertTable(rows){

			let getTextWidth = (str) => {
				let ret = str.length;
				for(let i=str.length; i--; ){
					if(str[i].charCodeAt(0) > 127){
						ret++;
					}
				}
				return ret;
			};

			let paddingRight = (str, char, length) => {
				for(var i=length-getTextWidth(str);i--;){
					str += char;
				}
				return str;
			};

			// 确定每一列的长度
			let colWidth = [];

			let rowArray = rows.map((row) => {
				let cols = row.split('\t');
				for(let i=0;i<cols.length;i++){
					let thisColWidth = getTextWidth(cols[i]);
					if(!colWidth[i]) colWidth[i] = 0;

					if(thisColWidth > colWidth[i]){
						colWidth[i] = thisColWidth;
					}
				}
				return cols;
			});

			// 添加一个表头下面的行（第2行）
			rowArray.splice(1, 0, colWidth.map((width)=>paddingRight('', '-', width)));

			let ret = rowArray.map((row) => {
				let ret = '|';

				ret += row.map((col, index) => {
					let thisColWidth = colWidth[index];
					return paddingRight(col, ' ', thisColWidth) + '|';
				}).join('');

				return ret;
			}).join('\n');

			let undo = _aceEditor.getSession().getUndoManager();

			setTimeout(()=>{
				undo.undo();
				_aceEditor.insert(`${ret}`);
			});

		},
		onEditorInput(){
			// logger.debug('onEditorInput');
			_value = _aceEditor.getValue();
			let cursorPosition = _aceEditor.getCursorPosition();
			let isEditingHeading = false;
			if(cursorPosition.row === 0){
				isEditingHeading = true;
			}
			// todo:isEditingHeading如何处理
			this.$emit('input', _value);
		},
		onKeydown($event){
			// meta(cmd/ctrl) + F
			if($event.keyCode === 70 && $event.metaKey){
				$event.preventDefault();
				$event.stopPropagation();
				this.isSearching = true;
				setTimeout(() => {
					this.$refs.findInput.focus();
				}, 400);
			}else if($event.keyCode === 27){
				if(this.isSearching){
					$event.preventDefault();
					$event.stopPropagation();
					this.isSearching = false;
					_aceEditor.focus();
					this.searchKeyword = '';
				}
			}
		},
		onSearch(){
			_aceEditor.find(this.searchKeyword);
		},
		resize(){
			_aceEditor.resize();
			// 动画完之后再resize一次
			setTimeout(()=>{
				_aceEditor.resize();
			},500);
		}
	},
	watch:{
		value(value){
			console.log('watch value change');
			if(!value && value !== '') return
			if(_value !== value){
				_aceEditor.setValue(value, -1);
				// 清除undo列表
				setTimeout(() => {
					_aceEditor.getSession().getUndoManager().reset();
				},0);
			}
		},
		/* tnEvent(data){
			switch(data.type){
				case 'newAttachment':
					this.insertAttachment(data);
					break;
				case 'layout':
					this.resize();
					break;
				case 'editAction':
					console.log('editAction', data);
					if(!data.action) return;
					let undo = _aceEditor.getSession().getUndoManager();
					if(data.action === 'undo') {
						if(undo.hasUndo()){
							undo.undo(true);
						}
					}else if(data.action === 'redo') {
						if(undo.hasRedo()){
							undo.redo(true);
						}
					}
					break;
			}
		}, */
	},
	data(){
		return {
			isSearching: false,
			searchKeyword: ''
		};
	},
	mounted(){
		var aceEditor = ace.edit('ace_container');
		var session = aceEditor.getSession();
		_aceEditor = aceEditor;
		aceEditor.setTheme('ace/theme/tomorrow');
		session.setMode('ace/mode/markdown');
		session.setUseWrapMode(true);
		aceEditor.renderer.setHScrollBarAlwaysVisible(false);
		aceEditor.renderer.setShowGutter(false);
		aceEditor.renderer.setPadding(20);
		aceEditor.setShowPrintMargin(false);
		aceEditor.$blockScrolling = Infinity;
		if(process.platform === 'win32'){
			// 设置字体后会导致宽度变大，挺奇怪的
			// aceEditor.setOption('fontFamily', '"PingFang SC", "Hiragino Sans GB", "Microsoft YaHei", "WenQuanYi Micro Hei", sans-serif');
		}
		aceEditor.on('input', this.onEditorInput);

		shortcut(aceEditor);
		/*for(let cmd in shortcut){
			aceEditor.commands.bindKey(cmd, shortcut[cmd]);
		}*/

		// 同步滚动
		session.on('changeScrollTop', throttle((scroll) => {
			let targetRow = aceEditor.getFirstVisibleRow();
			this.$emit('line-scroll', targetRow);
		}, 500));

		// 重新计算大小
		setTimeout(() => {
			this.resize();
		},0);

		window.addEventListener('resize', throttle(this.resize, 50));

		/* console.log(this.value, this);
		if(this.value){
			_aceEditor.setValue(this.value, -1);
		} */
	},
	destroyed(){
		_value = '';
		_aceEditor.destroy();
	}
};
</script>
<style scoped>
.editor{
	border-right:1px solid #E0E0E0;
	font-family: "PingFang SC", "Hiragino Sans GB", "Microsoft YaHei", "WenQuanYi Micro Hei", sans-serif;
	height:100%;
	flex:1;
	align-items: stretch;
	display: flex;
	flex-direction: column;
}
#ace_container{
	/* height:100%; */
	font-size: 14px;
    line-height: 28px;
	flex:1;
	/* display: flex; */
}
.find{
	height: 0;
	overflow: hidden;
	line-height: 28px;
	transition: height .4s;
}
.find.show{
	height: 28px;
}
.find .findInputWrapper{
	margin-right: 80px;
}
.find input{
	background:#F6F6F6;
	height: 27px;
    width: 100%;
    border: 0 none;
    vertical-align: top;
    padding: 0 10px;
	box-sizing: border-box;
	border-top:1px solid #e0e0e0;
}
.find input:focus{
	background:white;
	outline: 0 none;
}
.find button{
	width: 80px;
	float:right;
	border:0 none;
	height: 28px;
	background: #333;
	color:white;
	font-size:14px;
}
</style>
