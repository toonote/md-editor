# MdEditor

基于 Vue 的编辑器，用于[小兔笔记](https://xiaotu.io)。

## 安装

```sh
npm install git+https://github.com/TooNote/MdEditor.git
```

## 使用

封装为`.vue`单文件组件，直接引用使用。

### 通信

editor 会通过事件与外界通信，当内部有事件产生时，需要外部监听处理，目前会触发的事件：

- `save-image` 当粘贴或者拖拽图片时会触发，包含以下参数
    - `filepath` 本地文件的路径，或者`@clipboard`表示剪贴板
    - `ext` 文件后缀，如`.jpg`，可能为空

外界有事件需要内部响应时，可以修改`tnEvent`的值，`tnEvent`为一个对象

- `tnEvent`
    - `type` 事件类型
    - `data` 事件附加的数据

目前editor会响应如下类型事件：

- `imageUrl` 图片存储完毕，需要插入到编辑器中显示，`data.url`为图片地址
- `layout` 布局变更，需要 editor 重新适配大小

值得注意的点：

1. 文件未做预编译，需要使用者使用 `vue-loader`
2. 目前依赖 vuex （未来会改掉）
    - 编辑器中的内容改变时，会 dispatch `changeCurrentNoteContent`
    - 滚动时会 dispatch `syncScroll`
    - 依赖 getter ：`currentNote`, `editAction`

见如下示例代码：

```html
<template>
    <editor
        v-on:save-image="saveImage"
        v-bind:tn-event="tnEvent"
    ></editor>
</template>

<script>
    import editor from 'tn-md-editor';
    // 主app
    let app = new Vue({
        // 需要vuex store
        store,
        methods:{
            // 用于触发事件供editor响应
            _tnEvent: function(type, data){
                if(!data) data = {};
                this.tnEvent = {...data, type, _:Math.random()};
                this.$nextTick(() => {
                    // 触发后清空事件
                    this.tnEvent = {};
                });
            },
            // 编辑器中粘贴或者拖拽图片时响应
            saveImage: function(filepath, ext){
                if(filepath === '@clipboard'){
                    // 需要自己写方法存储图片并返回url
                    this._tnEvent('imageUrl', {url:io.saveImageFromClipboard()});
                }else{
                    // 需要自己写方法存储图片并返回url
                    this._tnEvent('imageUrl', {url:io.saveImage(filepath, ext)});
                }
            }
        },
        data:{
            // 编辑器会监听tnEvent，变动时响应事件
            tnEvent: {}
        },
        components: {
            editor
        }
    });
</script>
```