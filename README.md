# MdEditor

基于 Vue 的Markdown编辑器。

> 当前版本尚在开发中，API随时可能变动，请勿在生产环境中使用

## 安装

```sh
npm install @toonote/md-editor
```

## 使用

封装为`.vue`单文件组件，直接引用使用，支持双向绑定。文件未做预编译，需要使用者使用 `vue-loader`

见如下示例代码：

```html
<template>
    <editor
        ref="editor"
        v-model="content"
        v-on:attachment="saveAttachment"
        v-on:line-scroll="lineScroll"
    ></editor>
</template>

<script>
    import editor from 'tn-md-editor';
    // 主app
    let app = new Vue({
        methods:{
            // 编辑器中粘贴或者拖拽图片时响应
            saveAttachment: function(data){
                // 需要自己写方法存储附件并返回url，示例只存储第一个
                const result = saveAttachment(data.files[0]);
                this.$refs.editor.insertAttachment({
                    filename: result.filename,
                    ulr: result.url
                });
            },
            // 编辑器滚动
            lineScroll: function(row){
                // 滚动时做点什么事情，此处示例为触发vuex action
                console.log('do something', row);
            }
        },
        data:{
            // 编辑器的内容，当content变化时，编辑器内容也会变化
            content: ''
        },
        components: {
            editor
        }
    });
</script>
```

### 内容

内容采用双向绑定机制，指定`v-model`变量即可。

### 方法

通过`$refs`机制拿到组件实例后，可调用实例方法：

- `insertAttachment({filename:string, url:string})` 在编辑器光标位置插入附件（图片）
- `insertLink({title:string, url:string})` 在编辑器光标位置插入链接
- `layout` 布局变更，需要 editor 重新适配大小
- `editAction({action:string})` 编辑命令，`action`取值`undo` / `redo`

### 事件

组件会产生一些事件，当内部有事件产生时，需要外部监听处理，目前会触发的事件：

- `attachment` 当粘贴或者拖拽附件时会触发，参数：
    - `data` 参数对象
    - `data.files` 文件列表
    - `data.method` 文件来源，可能值为`drop`（表示拖拽）、`clipboard`（粘贴）
- `line-scroll` 滚动时触发，参数：
    - `row` 当前编辑器可视区域第一行行号

## 用户列表

- [小兔笔记](https://xiaoto.io)

## 历史记录

### 1.0.0 alpha 3 (2020-05-09)

- 添加插入链接方法`insertLink()`

### 1.0.0 alpha 2 (2020-02-15)

- 更新barce版本，解决safari下中文输入问题

### 1.0.0 alpha 1 (2019-08-20)

- 旧代码整理，适配浏览器环境使用
