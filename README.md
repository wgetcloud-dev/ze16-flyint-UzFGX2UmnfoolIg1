
#### 文件下载的几种方式。


大家都做过文件下载，无非就是通过a标签给定一个href。
用户点击下载按钮。
或者使用Blob的方式进行下载。
这两种是很常见的，也是我们平时做使用最多的方式。
那么我们知道这2种方式有什么区别呢？
如果不清楚，也别着急下面我们一起来探索下:


#### node \+ express \+ cors 搭建环境



```
// 引入express
const express=require("express");
// 引入路径模块
const path=require("path")
// 处理跨域的插件,需要下载一下这个模块
const cors = require('cors')
// 文件下载相关的接口 
const fileListRouter = require('./routes/fileList'); 
const app=express();
// 处理跨域
app.use(cors())
app.use(express.static(path.join(__dirname, '/public')));
// 下载文件的路由路径
app.use('/fileList', fileListRouter);

//服务端口
app.listen(3000,function () {
  console.log("127.0.0.1:3000")
});

```

#### 文件下载的思路分析


我们首先需要知道这个被下载文件的具体地址。
然后我们需要检查这个文件是否存在(假设存在就具有访问权限)。
如果存在的话，使用异步的方式进行读取。
然后给响应头设置文件类型、文件名、告诉浏览器是是接下载还是展示。
最后一步是把文件内容以二进制的形式写到响应体中，并发送出去。


#### node提供一个接口实现文件下载



```
const express = require('express');
const path = require('path');  
const fs = require('fs');
const router = express.Router();
router.get('/download', (req, res) => {
  // 获取传参对象信息
  const queryParams = req.query;
  console.log('传递的参数对象', queryParams)
  // 获取下载文件的名称,需要注意filename = 文件+文件类型，否者会出现404
  const filename = queryParams.fileName
  // 从当前所在的目录(__dirname)开始,进入到名为 allFIle 的子目录, 最后定位到名为 filename变量的文件
  const filePath = path.join(__dirname, 'allFIle', filename); 
  console.log('文件路径', filePath)
  // 检查文件是否存在  
  fs.access(filePath, fs.constants.F_OK, (err) => { 
    // 如果失败，err 是一个对象。成功的话，err是null
    if (err) {  
      return res.status(404).send('File not found');  
    } 
    // 'application/octet-stream':一种通用的二进制数据的MIME类型，表示 "任意二进制数据"。
    // 当我们不确定文件的MIME类型，或想要强制浏览器将响应作为文件下载而不是直接打开时，通常会使用这个类型。
    const mimeType = 'application/octet-stream';
    // 用于指示内容该以何种形式展示，直接在浏览器中打开(内联显示)还是作为附件下载 。
    // 这里的值 attachment; filename="${filename}"告诉浏览器将响应的内容作为附件下载，
    res.setHeader('Content-Disposition', `attachment; filename="${filename}"`);
    // 设置HTTP响应头Content-Type，表示响应的内容是任意二进制数据
    res.setHeader('Content-Type', mimeType);
    // 创建可读流并传递给响应对象
    const fileStream = fs.createReadStream(filePath); 
    fileStream.pipe(res);    
  });  
});  
module.exports = router;

```

#### 上面代码中的fs.access


在 Node.js 中，fs.access 是一个用于检查文件或目录是否存在
以及是否具有特定权限的异步文件系统方法。
该方法允许我们验证当前进程是否对文件或目录具有读取、写入或执行权限。


#### fs.access的基本语法



```
fs.access(path, mode, callback)
path: 文件或目录的路径
mode(可选):要检查的权限，可以使用或 | 运算符进行组合
  fs.constants.F_OK：检查文件是否存在。
  fs.constants.R_OK：检查是否具有读取权限。
  fs.constants.W_OK：检查是否具有写入权限。
  fs.constants.X_OK：检查是否具有执行权限。
如果没有提供 mode，则默认检查 fs.constants.F_OK
callback(Function)：回调函数，回调函数中有一个参数err。
如果操作成功，err 为 null；
如果操作失败，err 是一个 Error 对象。

```

#### 简单使用 fs.access



```
const fs = require('fs');
// 检查这个文件是否存在或者可读。
fs.access('./readme.md', fs.constants.F_OK | fs.constants.R_OK, (err) => {
  if (err) {
    console.error('文件不存在或不可读');
  } else {
    console.log('文件存在且可读');
  }
});

```

![](https://img2024.cnblogs.com/blog/1425695/202412/1425695-20241204204010133-1934884189.png)


#### 方式1:前端 a标签下载



```
<template>
  <div>
    <el-button>
      <a href='http://127.0.0.1:3000/fileList/download?fileName=shipin.mp4'>通过a标签直接下载a>
    el-button>
  div>
template>

```

#### 方式2:通过Blob的方式下载



```
<template>
  <div>
    <el-button type="primary" @click="fileDownHandler">
      通过blob来下载
    el-button>
  div>
template>
<script>
import axios from 'axios'
  export default {
    methods: {
      fileDownLoad(fileData, fileName, callBack) {
        // 创建Blob实例  fileData 接受的是一个Blob
        // 等待服务器把所有的数据都传输到浏览器的内存后,然后再把内存变为 blob 格式
        let blob = new Blob([fileData], {
          type: 'application/octet-stream',
        })
        if (!!window.ActiveXObject || 'ActiveXObject' in window) {
          window.navigator.msSaveOrOpenBlob(blob, fileName)
          callBack()
        } else {
          // 创建a标签
          const link = document.createElement('a')
          // 隐藏a标签
          link.style.display = 'none'
          // 在每次调用 createObjectURL() 方法时，都会创建一个新的 URL 指定源 object的内容
          // 或者说(link.href 得到的是一个地址，你可以在浏览器打开。指向的是文件资源)
          link.href = URL.createObjectURL(blob)
          console.log('link.href指向的是文件资源', link.href)
          //设置下载为excel的名称
          link.setAttribute('download', fileName)
          document.body.appendChild(link)
          // 模拟点击事件
          link.click()
          // 移除a标签
          document.body.removeChild(link)
          // 回调函数，表示下载成功
          callBack() 
        }
      },
      fileDownHandler(){
        axios.get('http://127.0.0.1:3000/fileList/download?fileName=shipin.mp4', { responseType: 'blob' }).then((response) => {
          console.log('文件下载返回来的数据', response)
          this.fileDownLoad(response.data, '视频文件', ()=>{
            console.log('下载成功')
          })
        }).catch(function (error) {
            console.log(error);
        });
      }
    }
  }
script>

```

![](https://img2024.cnblogs.com/blog/1425695/202412/1425695-20241204204030359-729822789.jpg)


#### a标签的下载方式：


数据从服务端不断流向浏览器
浏览器会不会等待服务器把这个文件的所有数据都传输完后再触发下载呢？
答案是:不会。等会我们可以通过来验证一下
浏览器只要确认这个响应是成功的。
它就不会去等待全部数据都传输过来，才触发下载。
而是直接去触发下载行为。
这样数据就像流水一般，从服务器经过浏览器流向了文件。
数据从服务器\=\=\>经过浏览器 \=\=\=\>文件。
这个过程浏览器不会保存这些数据
这样的好处：哪怕这个文件有很大（几十或者上百个G）对浏览器的内存都不会造成什么影响。
也就是说：数据经过浏览器流向文件（从网络到磁盘）这一过程对浏览器的内存几乎没怎么占用。
总结：通过a标签的形式来下载大文件是非常友好的。


![](https://img2024.cnblogs.com/blog/1425695/202412/1425695-20241204204041314-673479368.png)
![](https://img2024.cnblogs.com/blog/1425695/202412/1425695-20241204204048612-45756326.jpg)


#### blob的下载方式


浏览器会等待服务器把所有的数据都传输完成后。
把所有的数据都放入浏览器的内存中
然后生成一个blob对象
然后创建一个本地的url地址
创建a标签，通过a标签，下载保留在浏览器中的所有数据
这样会出现一个问题。
当文件较大的时候，会出现卡死的现象。
出现卡死的现象的原因：
浏览器会等待服务器中把所有的数据都传输完毕后，才进行下载。


![](https://img2024.cnblogs.com/blog/1425695/202412/1425695-20241204204137785-1358308835.jpg)


#### a标签和 Blob 下载的区别


1,在下载过程中，离开当前页或者属性页面。
a标签下载不会中断,会继续下载。Blob会中断下载。
2,在下载过程中,览器会不会等待服务器把该文件的所有数据都传输完后,才触发下载？
a标签下载不会等待所有的数据传输完毕后才触发下载，浏览器只要确认这个响应是成功的。就会马上触发下载。
Blob会等待所有的数据都传输完毕后才触发下载。
3,a标签的下载无法携带token进行鉴权(但可以通过cookie鉴权)，Blob可以进行鉴权


#### axios中的onDownloadProgress函数的出场


在做文件下载的时候
如果文件较大产品希望可以显示一个进度条
那么axios支持吗？
通过查看文档,它有一个onDownloadProgress函数
用于获取请求的进度结果
但是需要后端配合,在响应头中设置Content\-Length属性,
然后再onDownloadProgress函数中两个非常重要的字段total和loaded
total表示当前文件的大小,单位是byte(字节)
loaded表示当前获取的文件进度
然后我们就能计算出当前文件的下载进度了
如果不设置Content\-Length,则total的值是0或者undefined


![](https://img2024.cnblogs.com/blog/1425695/202412/1425695-20241204204152429-1263101215.png)
![](https://img2024.cnblogs.com/blog/1425695/202412/1425695-20241204204157666-739901369.png)


#### node提供文件下载的接口并设置Content\-Length



```
router.get('/downFile', (req, res) => {
  const filename = 'shipin.mp4'
  // 从当前所在的目录(__dirname)开始,进入到名为 allFIle 的子目录, 最后定位到名为 filename变量的文件
  const filePath = path.join(__dirname, 'allFIle', filename); 
  console.log('文件路径', filePath)
  // 一个异步方法，用于获取文件或目录的详细信息，包括文件大小、创建时间、修改时间、权限等。  
  fs.stat(filePath,  (err, stats) => { 
    // 如果失败，err 是一个对象。成功的话，err是null
    if (err) {  
      return res.status(404).send('File not found');  
    } 
    const fileSize = stats.size;
    console.log('fileSize', fileSize)
    // 文件类型
    const mimeType = 'video/mp4';
    res.setHeader('Content-Disposition', `attachment; filename="${filename}"`);
    // 设置HTTP响应头Content-Type，表示响应的内容是视频
    res.setHeader('Content-Type', mimeType);
    // 设置文件大小，如果要获取响应进度，这个属性必须设置
    res.setHeader('Content-Length', fileSize);
    // 创建可读流并传递给响应对象
    const fileStream = fs.createReadStream(filePath); 
    fileStream.pipe(res);    
  });  
});

```

#### onDownloadProgress函数的使用



```
export function fileDownShowProgress(params, callBack) {
  return httpRequest({
    method: 'get',
    url: '/fileList/downFile',
    baseURL: 'http://127.0.0.1:3000',
    isAbort: true, // 这个属性是我封装的取消请求，可以忽略
    params: params,
    responseType: 'blob',
    onDownloadProgress: function (progressEvent) {
      console.log('progressEvent参数:', progressEvent)
      // 计算下载进度百分比
      const percentNum = Math.round((progressEvent.loaded * 100) / progressEvent.total);
      callBack(percentNum)
    }
  })
}

```


```
<template>
  <div class="down-page">
    <el-button @click="fileDownShowProgressApi">文件下载-显示进度el-button>
    <div class="set-with" v-if="hiddenFlag">
      <el-progress :percentage="percentNum">el-progress>
    div>
  div>
template>
<script>
import {fileDownShowProgress} from '@/request/api.js'
export default {
  data(){
    return {
      percentNum:0,
      hiddenFlag:false
    }
  },
  methods:{
    fileDownShowProgressApi(){
      fileDownShowProgress({},(percentNum)=>{
        console.log('进度值', percentNum)
        this.percentNum = percentNum
        if(this.percentNum <= 0 || this.percentNum >=100){
          this.hiddenFlag = false
        }else{
          this.hiddenFlag = true
        }
      }).then(res=>{
        console.log('返回来的数据', res)
        this.fileDownLoad(res, '视频文件', ()=>{
          console.log('下载成功')
        })
      }).catch(err=>{
        // 有可能超时或其他异常情况
        this.hiddenFlag= false
        this.percentNum = 0
        console.log('err:', err)
      })
    },

    fileDownLoad(fileData, fileName, callBack) {
        ...省略上面有这一部分的代码...
        callBack() 
      }
    },
  }
}

```

![](https://img2024.cnblogs.com/blog/1425695/202412/1425695-20241204204305220-1910363603.png)
![](https://img2024.cnblogs.com/blog/1425695/202412/1425695-20241204204312230-250545801.png)


 本博客参考[wgetcloud全球加速器服务](https://wgetcloud6.org)。转载请注明出处！
