# MdEditor

基于 Vue 的编辑器，用于[小兔笔记](https://xiaotu.io)。

## 安装

```sh
npm install git+https://github.com/TooNote/MdEditor.git
```

## 使用

封装为`.vue`单文件组件，直接引用使用。

值得注意的点：

1. 文件未做预编译，需要使用者使用 `vue-loader`
2. 编辑器中可能粘贴图片或者拖拽插图，需要在 app 中自己监听事件处理图片存储，并改变`this.imageUrl`
3. 目前依赖 vuex （未来会改掉）
    - 编辑器中的内容改变时，会 dispatch `changeCurrentNoteContent`
    - 滚动时会 dispatch `syncScroll`
    - 依赖 getter ：`currentNote`, `layout`, `editAction`

见如下示例代码：

```html
<template>
    <editor
        v-on:save-image="saveImage"
        v-bind:image-url="imageUrl"
    ></editor>
</template>

<script>
    import editor from 'tn-md-editor';
    // 主app
    let app = new Vue({
        // 需要vuex store
        store,
        methods:{
            // 编辑器中粘贴或者拖拽图片时响应
            saveImage: function(filepath, ext){
                if(filepath === '@clipboard'){
                    // 需要自己写方法存储图片并返回url
                    this.imageUrl = io.saveImageFromClipboard();
                }else{
                    // 需要自己写方法存储图片并返回url
                    this.imageUrl = io.saveImage(filepath, ext);
                }
                console.log(this.imageUrl);
            }
        },
        data:{
            // 编辑器会监听imageUrl，变动时在编辑器中插入图片
            imageUrl: ''
        },
        components: {
            editor
        }
    });
</script>
```