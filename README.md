# 文件上传（支持分片上传、断点续传、秒传、暂停/恢复、进度条等）

## 背景

在我们的项目开发中基本上都会有上传文件的需求。通常我们使用 ajax + formData 的方式将文件上传到指定的服务器上。出于对存储压力的考虑，一般上传的文件也都不会很大，会限制在几M ~ 几十M以内。

然而，上传大文件的需求肯定会存在的，比如视频上传业务。一个视频可能高达好几个G，那么就必须要面临这样的问题：一是上传时间过长，中间一旦发生错误就会导致整个上传任务失败；二是服务端处理起来也很麻烦，要配置超大表单的处理、超时的问题等等。

这时候，比较好的方式是前端在上传大文件时，将文件进行切片，分片上传到服务器，再由服务端进行合并。这样做可以避免一旦发生错误需要整个文件重传，同时，服务端也简化了复杂的配置。

那么，下面我们就来实现一下支持分片上传、断点续传、秒传、暂停/恢复、进度条等功能的文件上传功能。

附项目地址：[upload-files](https://github.com/zhangyanling77/upload-files)

## 前端部分

> 技术栈：react + antd 3.x + typescript

### 创建项目

> 其中高级配置可按照antd官网来进行配置，这里不做赘述

```bash
mkdir upload-files && cd upload-files
npx create-react-app client --template typescript
cd client
yarn add antd react-app-rewired customize-cra babel-plugin-import ...
yarn start
```

`App.tsx`

> 引入自己封装的 Upload 组件

```tsx
import React, { Component } from 'react';
// ...
import Upload from './Upload';

class App extends Component {
  render() {
    return (
      <div className="App">
        <Upload />
      </div>
    );
  }
}
export default App;
```

`Upload.tsx`

> 由于代码内容过长，这里拣重点做说明，具体代码实现可参考项目源码。包含了文件上传的校验、文件的切片、文件的预览、通过计算文件的hash值实现缓存（秒传的实现）、上传的暂停/恢复、上传进度展示等。

- 上传组件基础布局

  左侧根据上传状态展示上传、暂停、恢复等按钮，右侧为图片预览区域，包含图片展示及文件名显式，最下面展示上传进度组件。

```tsx
<Fragment>
// ...
<Card>
  <Input type='file' style={{ width: 300 }} onChange={handleChange} />
  {
    uploadStatus === UploadStatus.INIT &&
    <Button type="primary" onClick={handleUpload} style={{ marginLeft: 10 }}>上传</Button>
  }
  {
    uploadStatus === UploadStatus.UPLOADING &&
    <Button type="primary" onClick={handlePause} style={{ marginLeft: 10 }}>暂停</Button>
  }
  {
    uploadStatus === UploadStatus.PAUSE &&
    <Button type="primary" onClick={handleResume} style={{ marginLeft: 10 }}>恢复</Button>
  }
</Card>
// ...
<Card
  title="预览效果"
  hoverable
  cover={objectURL && <img alt='' src={objectURL} />}
>
  文件名：{currentFile && currentFile.name}
</Card>

// ...
{uploadProgress}
</Fragment>
```

- 定义文件切片的大小

这个大小可以根据自己项目的实际需求来定。

```ts
const DEFAULT_SIZE = 1024 * 1024 * 100; // 按100M切片
```

- 定义文件上传状态的枚举

> INIT：未上传的初始状态 \
> PAUSE：暂停 \
> UPLOADING：上传中

```ts
enum UploadStatus {
  INIT,
  PAUSE,
  UPLOADING
}
```
- 定义分片内容接口和上传的内容接口

```ts
// 分片的内容
interface Part {
  chunk: Blob; // 分割出来的块
  size: number; // chunk大小
  filename?: string; // 文件名
  chunk_name?: string; // 块的名字
  loaded?: number; // 已经上传的字节数
  percent?: number; // 加载的百分比
  xhr?: XMLHttpRequest; // xhr对象，可控制请求的发送和中断
}
// 上传的内容
interface Uploaded {
  filename: string; // 文件名
  size: number; // 文件大小
}
```

- 定义文件上传的相关状态

```ts
const [uploadStatus, setUploadStatus] = useState<UploadStatus>(UploadStatus.INIT); // 上传状态的控制
const [currentFile, setCurrentFile] = useState<File>(); // 当前上传的文件
const [objectURL, setObjectURL] = useState<string>(''); // 图片预览的url设置
const [hashPercent, setHashPercent] = useState<number>(0); // 上传完成的百分比
const [filename, setFilename] = useState<string>(''); // 文件名设置
const [partList, setPartList] = useState<Part[]>([]); // 分片的文件列表
```

- 选择文件

```ts
function handleChange(event: ChangeEvent<HTMLInputElement>) {
  const file: File = event.target.files![0];
  setCurrentFile(file);
}
```

- 图片预览

  `FileReader` 对象允许Web应用程序异步读取存储在用户计算机上的文件（或原始数据缓冲区）的内容。使用 `File` 或 `Blob` 对象指定要读取的文件或数据，读取完成将会包含一个data:URL格式的base64字符串表示读取的文件内容。

```ts
useEffect(() => {
  if (currentFile) {
    const reader = new FileReader();
    reader.readAsDataURL(currentFile);
    reader.addEventListener('load', () => setObjectURL(reader.result as string));
  }
}, [currentFile]);
```

- 上传校验

```ts
function beforeUpload(file: File) {
  // 文件类型白名单
  const isValidFileType = ["image/jpeg", "image/png", "image/gif", "video/mp4"].includes(file.type);
  if (!isValidFileType) {
    message.error('不支持此类文件上传');
  }
  // 文件大小限制
  const isLessThan2G = file.size < 1024 * 1024 * 1024 * 2;
  if (!isLessThan2G) {
      message.error('上传的文件不能大于2G');
  }
  return isValidFileType && isLessThan2G;
}
```

- 创建分片

```ts
// handleUpload 上传按钮
const partList:Part[] = createChunks(currentFile);
// ...
const lastDotIndex = currentFile.name.lastIndexOf('.')
const extName = currentFile.name.slice(lastDotIndex)
const filename = `${fileHash}${extName}`
setFilename(filename); // 设置文件名
partList.forEach((item:Part, index:number) => {
  item.filename = filename;
  item.chunk_name = `${filename}-${index}`; //分块的名称
  item.loaded = 0;
  item.percent = 0;
})
setPartList(partList); // 分片数组
// ...
function createChunks(file:File):Part[]{
    let current = 0;
    const partList:Part[] = [];
    while(current < file.size) {
      const chunk = file.slice(current, current + DEFAULT_SIZE)
      partList.push({
        chunk,
        size: chunk.size
      })
      current += DEFAULT_SIZE
    }
    return partList
  }
```

- 计算文件hash做缓存

```ts
// handleUpload 上传按钮
// 先计算这个对象的hash值 秒传的功能
const fileHash:string = await calculateHash(partList) as string;
// ...
function calculateHash(partList:Part[]) {
  return new Promise((resolve, reject) => {
    const worker = new Worker('/hash.js')
    worker.postMessage({ partList })
    worker.onmessage = (event) => {
      const { percent, hash } = event.data; // 获取百分比和hash值
      setHashPercent(percent);
      if (hash) {
        resolve(hash);
      }
    };
  })
}
```

`hash.js` 放在根目录/public下

> 通过webworker子进程来计算hash

```js
/* eslint-disable no-restricted-globals */
self.importScripts('https://cdn.bootcss.com/spark-md5/3.0.0/spark-md5.js'); // 引入MD5
// 监听消息
self.onmessage(async (event) => {
  const { partList } = event.data; // 获取传过来的分片数组
  const spark = new self.SparkMD5.ArrayBuffer(); // 生成MD5码
  let percent = 0;
  const perSize = 100 / partList.length;
  const buffers = await Promise.all(partList.map(({ chunk }) => new Promise((resolve) => {
    const reader = new FileReader();
    reader.readAsArrayBuffer(chunk);
    reader.onload = (event) => {
      percent += perSize;
      self.postMessage({ percent: Number(percent.toFixed(2)) });
      resolve(event.target.result);
    }
  })));
  buffers.forEach(bf => spark.append(bf));
  self.postMessage({ percent: 100, hash: spark.end() });
  self.close();
})
```

- 上传分片

  先去后端验证当前上传的文件是否还需要上传，如果已经存在，那么秒传成功；否则进行分片上传，并比较服务端已经传了的内容和当前要传的内容，是否需要断点续传。当所有的分片上传成功后，再向服务端发起分片合并的请求。

```ts
// 根据文件名去服务端校验文件是否已经上传过了
async function verify(filename: string) {
  return await request({
    url: `/verify/${filename}`,
  })
}

async function uploadParts(partList: Part[], filename: string) {
  // 校验是否已经上传过
  const { needUpload, uploadList } = await verify(filename);
  if (!needUpload) {
    return message.success('秒传成功');
  }
  try {
    const requests = createRequests(partList, uploadList, filename);
    await Promise.all(requests);
    // 合并分片请求
    await request({ url: `/merge/${filename}` });
    message.success('上传成功');
    reset();
  } catch (error) {
    message.error('上传失败或暂停');
  }
}
// 分片上传逻辑
function createRequests(partList:Part[], uploadList:Uploaded[], filename:string) {
  return partList.filter((part: Part) => {
    const uploadFile = uploadList.find(item => item.filename === part.chunk_name);
    // 完全没有上传过
    if (!uploadFile) {
      part.loaded = 0; // 已经上传的字节数0
      part.percent = 0; // 已经上传的百分比就是0 分片的上传过的百分比
      return true;
    }
    // 之前上传了部分内容，断点续传逻辑
    if (uploadFile.size < part.chunk.size) {
      part.loaded = uploadFile.size;// 已经上传的字节数
      part.percent = Number((part.loaded / part.chunk.size * 100).toFixed(2));//已经上传的百分比
      return true;
    }
    return false;
  }).map((part: Part) => request({
    url: `/upload/${filename}/${part.chunk_name}/${part.loaded}`,// 请求的URL地址
    method: 'POST', // 请求的方法
    headers: { 'Content-Type': 'application/octet-stream' }, // 指定请求体的格式
    setXHR: (xhr: XMLHttpRequest) => part.xhr = xhr, // 为了控制请求中断
    onProgress: (event: ProgressEvent) => {
      part.percent = Number(((part.loaded! + event.loaded) / part.chunk.size * 100).toFixed(2));
      setPartList([...partList]);
    },
    data: part.chunk.slice(part.loaded) // 请求体
  }))
}
```

- 请求方法封装，支持中断和请求进度

```ts
interface OPTIONS {
  baseURL?: string;
  method?: string;
  url: string;
  headers?: any;
  data?: any;
  setXHR?: any;
  onProgress?: any;
}

export default function request(options: OPTIONS):Promise<any> {
  const defaultOptions = {
    method: 'GET',
    baseURL: 'http://localhost:4000', // 服务端host
    headers: {},
    data: {},
  }
  options = {...defaultOptions, ...options, headers: { ...defaultOptions.headers, ...(options.headers || {}) } }
  return new Promise(function(resolve:Function, reject:Function) {
    const xhr = new XMLHttpRequest()
    xhr.open(options.method!, options.baseURL + options.url)
    // 设置请求头
    for (let key in options.headers) {
        xhr.setRequestHeader(key, options.headers[key]);
    }
    xhr.responseType = 'json';
    xhr.upload.onprogress = options.onProgress; // 上传进度
    xhr.onreadystatechange = function () {
      if (xhr.readyState === 4) {
        if (xhr.status === 200) {
            resolve(xhr.response);
        } else {
            reject(xhr.response);
        }
      }
    }
    if (options.setXHR) {
      options.setXHR(xhr); // 方便操作请求
    }
    xhr.send(options.data)
  })
}
```

- 暂停/恢复

```ts
async function handlePause() {
  // 中断上传
  partList.forEach((part: Part) => part.xhr && part.xhr.abort());
  setUploadStatus(UploadStatus.PAUSE);
}
async function handleResume() {
  setUploadStatus(UploadStatus.UPLOADING);
  // 再次上传分片
  await uploadParts(partList, filename);
}
```

- 进度条展示

```tsx
// ...
const columns = [
  {
    title: '切片名称',
    dataIndex: "filename",
    key: 'filename',
    width: '20%'
  },
  {
    title: '进度',
    dataIndex: "percent",
    key: 'percent',
    width: '80%',
    render: (value: number) => {
        return <Progress percent={value} />
    }
  }
];
// 总进度百分比
const totalPercent = partList.length > 0 ? partList.reduce(
  (acc: number, curr: Part) => acc + curr.percent!, 0) / (partList.length * 100) * 100
  : 0;
const uploadProgress = uploadStatus !== UploadStatus.INIT ? (
  <Fragment>
    <Row>
      <Col span={4}>
        HASH总进度:
      </Col>
      <Col span={20}>
        <Progress percent={hashPercent} />
      </Col>
    </Row>
    <Row>
      <Col span={4}>
        总进度:
      </Col>
      <Col span={20}>
        <Progress percent={totalPercent} />
      </Col>
    </Row>
    <Table
      columns={columns}
      dataSource={partList}
      rowKey={row => row.chunk_name!}
    />
  </Fragment>
) : null;
```

## 后端部分

> 技术栈： express + typescript

创建项目

```bash
mkdir server && cd server
npm init -y
npx tsconfig.json
yarn add fs-extra express morgan http-errors http-status-codes cors multer
yarn add @types/node cross-env typescript ts-node ts-node-dev nodemon -D
```

- 添加启动项目的script

```json
"scripts": {
  "dev": "cross-env PORT=4000 nodemon --exec ts-node --files ./src/www.ts"
}
```

`www.ts`

启动服务，进行监听

```ts
import app from './app';
import http from 'http';

const port = process.env.PORT || 4000;
const server = http.createServer(app);

server.listen(port);
server.on('error', onError);
server.on('listening', onListening);
function onError(error: any) {
    console.error(error);
}
function onListening() {
    console.log('Listening on ' + port);
}
```

`app.ts`

- 上传分片处理

判断是否存在没传完的临时目录，若存在则断点续传，否则创建临时目录存放分片。

```ts
// ...依赖引入
import { UPLOAD_DIR, TEMP_DIR, mergeChunks } from './utils';
// ...
app.post('/upload/:filename/:chunk_name/:start', async function (req: Request, res: Response, _next: NextFunction) {
  const { filename, chunk_name } = req.params
  const start:number = Number(req.params.start)
  const chunk_dir = path.resolve(TEMP_DIR, filename)
  const exist = await fs.pathExists(chunk_dir)
  if (!exist) {
    await fs.mkdirs(chunk_dir)
  }
  const chunkFilePath = path.resolve(chunk_dir, chunk_name)
  // 可写流
  const ws = fs.createWriteStream(chunkFilePath, { start, flags: 'a' })
  req.on('end', () => {
    ws.close()
    res.json({ success: true })
  })
  req.on('error', () => {
    ws.close()
  })
  req.on('close', () => {
    ws.close()
  })
  req.pipe(ws)
})
```

- 合并分片

```ts
// app.ts
app.get('/merge/:filename', async function (req: Request, res: Response) {
  const { filename } = req.params;
  await mergeChunks(filename);
  res.json({ success: true });
})

// utils.ts
// ...其他依赖
const DEFAULT_SIZE = 1024 * 1024 * 100;
export const UPLOAD_DIR = path.resolve(__dirname, 'upload'); // 上传目录
export const TEMP_DIR = path.resolve(__dirname, 'temp'); // 分片存放临时目录
// ...
const pipeStream = (filePath:string, ws:WriteStream) => new Promise((resolve, _reject) => {
  const rs = fs.createReadStream(filePath); // 可读流
  rs.on('end', async () => {
    await fs.unlink(filePath); // 读取完毕后删除文件
    resolve()
  })
  rs.pipe(ws)
})
// ...
export const mergeChunks = async (filename: string, size: number = DEFAULT_SIZE) => {
  const filePath = path.resolve(UPLOAD_DIR, filename)
  const chunkDir = path.resolve(TEMP_DIR, filename)
  const chunkFiles = await fs.readdir(chunkDir) // 读取所有目录下的文件
  // 按文件名进行升序排序
  chunkFiles.sort((a, b) => Number(a.split('-')[1] - Number(b.split('-')[1])));
  // 并行写入分片
  await Promise.all(chunkFiles.map((chunkFile:string, index:number) => pipeStream(
    path.resolve(chunkDir, chunkFile),
    fs.createWriteStream(filePath, {
      start: size * index
    })
  )))
  await fs.rmdir(chunkDir) // 清空临时目录下的文件
}
```

- 验证文件是否已经上传过

  通过文件名判断文件是否已经上传过（完毕），若已经上传完则不必再传。然后再盼临时目录下是否存在该文件，存在说明没传完，需要前端执行断点续传逻辑，并将已经传好的分片返回前端进行对比。

```ts
app.get('/verify/:filename', async (req: Request, res: Response): Promise<any> => {
  const { filename } = req.params;
  const filePath = path.resolve(UPLOAD_DIR, filename);
  const existFile = await fs.pathExists(filePath);
  if (existFile) {
    return {
      success: true,
      needUpload: false // 因为已经上传过了，所以不再需要上传了，可以实现秒传
    }
  }
  const tempDir = path.resolve(TEMP_DIR, filename);
  const exist = await fs.pathExists(tempDir);
  let uploadList: any[] = [];
  if (exist) {
    uploadList = await fs.readdir(tempDir);
    uploadList = await Promise.all(uploadList.map(async (filename: string) => {
      const stat = await fs.stat(path.resolve(tempDir, filename));
      return {
        filename,
        size: stat.size, // 现在的文件大写 100M  30M
      }
    }));
  }
  res.json({
    success: true,
    needUpload: true,
    uploadList, // 已经上传的文件列表
  });
});
```

## 结语

通过这个小项目的实践，我们可以知道，对于大文件的分片上传、断点续传是需要前后端配合来实现的。大文件按照分片规格在前端进行切片后传到后端，后端将切片先传到临时目录，等到切片传完，前端发起合并请求，后端对临时目录下的文件进行合并拼接后通过流的方式写入到上传目录下，即完成了分片上传。若上传过程中发生了错误，或者前端对上传操作进行了暂停，在后端临时目录下会保存已经上传的分片，当恢复上传后，后端会告诉前端文件上传的进度，已经上传的分片。前端再通过判断后，继续后续的上传，即断点续传的原理实现。
