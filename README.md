# upload-files

## 文件上传，实现分片上传、断点续传、秒传、暂停/恢复、进度条等功能
### 前端部分（create-reac-app + antd + typescript)
* 创建项目并安装依赖
```bash
$ mkdir upload-project && cd upload-project
$ npx create-react-app client --tempalte typescript
$ yarn add antd
$ cd client
$ yarn start
```
高级配置可按照antd官方来进行配置，这里不做赘述。

* 修改App.tsx
```javascript
  import React, { Component } from 'react';
  import './App.less';
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
* Upload.tsx
```javascript
  import React, { ChangeEvent, useState, useEffect } from 'react';
  import { Row, Col, Input, Button, Card, message, Table, Progress } from 'antd';
  import { request } from './request';
  const DEFAULT_SIZE = 1024 * 1024 * 100; // 按100M切片
  enum UploadStatus {
    INIT,
    PAUSE,
    UPLOADING
  }
  interface Part {
    chunk: Blob;
    size: number;
    filename?: string;
    chunk_name?: string;
    loaded?: number;
    percent?: number;
    xhr?: XMLHttpRequest;
  }
  interface Uploaded {
    filename: string;
    size: number;
  }
  // 上传文件校验
  function beforeUpload(file: File) {
    const isValidFileType = ["image/jpeg", "image/png", "image/gif", "video/mp4"].includes(file.type);
    if (!isValidFileType) {
      message.error('不支持此类文件上传');
    }
    // 文件大小的单位是字节  1024 bytes = 1 k * 1024 = 1 M * 1024 = 1 G
    const isLessThan2G = file.size < 1024 * 1024 * 1024 * 2;
    if (!isLessThan2G) {
        message.error('上传的文件不能大于2G');
    }
    return isValidFileType && isLessThan2G;
  }
  // 文件切片
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
  
  function Upload() {
      const [uploadStatus, setUploadStatus] = useState<UploadStatus>(UploadStatus.INIT);
      const [currentFile, setCurrentFile] = useState<File>();
      const [objectURL, setObjectURL] = useState<string>('');
      const [hashPercent, setHashPercent] = useState<number>(0);
      const [filename, setFilename] = useState<string>('');
      const [partList, setPartList] = useState<Part[]>([]);
      
      useEffect(() => {
        if (currentFile) {
          const reader = new FileReader();
          reader.addEventListener('load', () => setObjectURL(reader.result as string));
          reader.readAsDataURL(currentFile);
        }
      }, [currentFile]);
      
      function handleChange(event: ChangeEvent<HTMLInputElement>) {
        const file: File = event.target.files![0];
        setCurrentFile(file);
      }
      // 计算hash
      function calculateHash(partList:Part[]) {
        return new Promise((resolve, reject) => {
          const worker = new Worker('/hash.js')
          worker.postMessage({ partList })
          worker.onmessage = (event) => {
            const { percent, hash } = event.data;
            setHashPercent(percent);
            if (hash) {
              resolve(hash);
            }
          };
        })
      }
      function reset() {
        setUploadStatus(UploadStatus.INIT)
        setHashPercent(0)
        setPartList([])
        setFilename('')
      }
      async function handleUpload() {
        if (!currentFile) {
            return message.error('你尚未选择文件');
        }
        if (!beforeUpload(currentFile)) {
            return message.error('不支持此类文件的上传');
        }
        setUploadStatus(UploadStatus.UPLOADING);
        // 方式1: 这种方式是文件整个一起上传的
        // const formData = new FormData();
        // formData.append('chunk', currentFile);//添加文件，字段名chunk
        // formData.append('filename', currentFile.name); // 文件名
        // const result = await request({
        //   url: '/upload',
        //    method: 'POST',
        //   data: formData
        // });
        // message.info('上传成功');
        // 方式2: 实现分片上传
        const partList:Part[] = createChunks(currentFile)
        // 先计算这个对象的hash值 秒传的功能 通过webworker子进程来计算hash
        const fileHash:string = await calculateHash(partList) as string
        const lastDotIndex = currentFile.name.lastIndexOf('.')
        const extName = currentFile.name.slice(lastDotIndex)
        const filename = `${fileHash}${extName}`
        setFilename(filename);
        partList.forEach((item:Part, index:number) => {
          item.filename = filename; // 文件名
          item.chunk_name = `${filename}-${index}`; //分块的名称
          item.loaded = 0;
          item.percent = 0;
        })
        setPartList(partList);
        await uploadParts(partList, filename)
      }
      async function verify(filename: string) {
        return await request({
          url: `/verify/${filename}`,
        })
      }
      async function uploadParts(partList: Part[], filename: string) {
        const { needUpload, uploadList } = await verify(filename)
        if (!needUpload) {
          return message.success('秒传成功');
        }
        try {
          const requests = createRequests(partList, uploadList, filename);
          await Promise.all(requests);
          await request({ url: `/merge/${filename}` });
          message.success('上传成功');
          reset();
        } catch (error) {
            message.error('上传失败或暂停');
        }
      }
      function createRequests(partList:Part[], uploadList:Uploaded[], filename:string) {
        return partList.filter((part: Part) => {
          const uploadFile = uploadList.find(item => item.filename === part.chunk_name);
          if (!uploadFile) {
            part.loaded = 0; // 已经上传的字节数0
            part.percent = 0; // 已经上传的百分比就是0 分片的上传过的百分比
            return true;
          }
          if (uploadFile.size < part.chunk.size) {
            part.loaded = uploadFile.size;// 已经上传的字节数
            part.percent = Number((part.loaded / part.chunk.size * 100).toFixed(2));//已经上传的百分比
            return true;
          }
          return false;
        }).map((part: Part) => request({
          url: `/upload/${filename}/${part.chunk_name}/${part.loaded}`,//请求的URL地址
          method: 'POST', // 请求的方法
          headers: { 'Content-Type': 'application/octet-stream' }, // 指定请求体的格式
          setXHR: (xhr: XMLHttpRequest) => part.xhr = xhr,
          onProgress: (event: ProgressEvent) => {
              part.percent = Number(((part.loaded! + event.loaded) / part.chunk.size * 100).toFixed(2));
              console.log('part.percent', part.chunk_name, part.percent);
              setPartList([...partList]);
          },
          data: part.chunk.slice(part.loaded) // 请求体
        }))
      }
      async function handlePause() {
        partList.forEach((part: Part) => part.xhr && part.xhr.abort());
        setUploadStatus(UploadStatus.PAUSE);
      }
      async function handleResume() {
        setUploadStatus(UploadStatus.UPLOADING);
        await uploadParts(partList, filename);
      }
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
      ]
      const totalPercent = partList.length > 0 ? partList.reduce(
        (acc: number, curr: Part) => acc + curr.percent!, 0) / (partList.length * 100) * 100
        : 0
      const uploadProgress = uploadStatus !== UploadStatus.INIT ? (
          <>
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
          </>
      ) : null;
      
      return (
        <>
           <Row>
              <Col span={12} style={{ padding: 20 }}>
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
              </Col>
              <Col span={12} style={{ padding: 20 }}>
                <Card
                  title="预览效果"
                  hoverable
                  cover={objectURL && <img alt='' src={objectURL} />}
                >
                  {currentFile && currentFile.name}
                </Card>
              </Col>
          </Row>
          {uploadProgress}
      </>
      )
  }

  export default Upload;
```
* hash.js 放在根目录/public下
```javascript
  /* eslint-disable no-restricted-globals */
  self.importScripts('https://cdn.bootcss.com/spark-md5/3.0.0/spark-md5.js');
  self.onmessage(async (event) => {
    const { partList } = event.data
    const spark = new self.SparkMD5.ArrayBuffer();
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
* request.tsx
```javascript
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
    for (let key in options.headers) {
        xhr.setRequestHeader(key, options.headers[key]);
    }
    xhr.responseType = 'json';
    xhr.upload.onprogress = options.onProgress;
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
      options.setXHR(xhr);
    }
    xhr.send(options.data)
  })
}
```

### 后端部分（express + typescript）
* 创建项目并安装依赖
```bash
$ mkdir server && cd server
$ npm init -y
$ npx tsconfig.json
$ yarn add fs-extra express morgan http-errors http-status-codes cors multer multiparty
$ yarn add @types/node cross-env typescript ts-node ts-node-dev nodemon -D
```
* 添加启动项目的script
```json
"scripts": {
  "dev": "cross-env PORT=4000 nodemon --exec ts-node --files ./src/www.ts",
  "utils": "ts-node ./src/utils.ts"
}
```
* app.ts
```javascript
import express, { Request, Response, NextFunction } from 'express';
import logger from 'morgan';
import { INTERNAL_SERVER_ERROR } from 'http-status-codes'; // 500
import createError from 'http-errors';
import cors from 'cors';
import path from 'path';
import fs from 'fs-extra';
// import multiparty from 'multiparty'; // 处理文件上传
import { UPLOAD_DIR, TEMP_DIR, mergeChunks } from './utils'

const app = express()

app.use(logger('dev'))
app.use(express.json())
app.use(express.urlencoded({ extended: true }))
app.use(cors())
app.use(express.static(UPLOAD_DIR))

app.post('/upload/:filename/:chunk_name/:start', async function (req: Request, res: Response, _next: NextFunction) {
  const { filename, chunk_name } = req.params
  const start:number = Number(req.params.start)
  const chunk_dir = path.resolve(TEMP_DIR, filename)
  const exist = await fs.pathExists(chunk_dir)
  if (!exist) {
    await fs.mkdirs(chunk_dir)
  }
  const chunkFilePath = path.resolve(chunk_dir, chunk_name)
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
// 合并
app.get('/merge/:filename', async function (req: Request, res: Response) {
  const { filename } = req.params;
  await mergeChunks(filename);
  res.json({ success: true });
})

//每次先计算hash值 
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
// 上传文件
// app.post('/upload', function(req:Request, res:Response, next:NextFunction) {
//   const form = new multiparty.Form()
//   form.parse(req, async (err:any, fields, files) => {
//     if (err) {
//       return next(err)
//     }
//     const filename = fields.filename[0]
//     const chunk = files.chunk[0]
//     await fs.move(chunk.path, path.resolve(UPLOAD_DIR, filename), { overwrite: true })
//     res.json({ 
//       success: true
//     });
//   })
// })
app.use(function(_req:Request, _res:Response, next:NextFunction) {
  next(createError(404))
})

app.use(function(error:any, _req:Request, res:Response, _next:NextFunction) {
  res.status(error.status || INTERNAL_SERVER_ERROR)
  res.json({
    success: false,
    error
  })
})

export default app
```
* www.ts
```javascript
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
* utils.tsx
```javascript
import path from 'path';
import fs, { WriteStream } from 'fs-extra';
const DEFAULT_SIZE = 1024 * 1024 * 100;
export const UPLOAD_DIR = path.resolve(__dirname, 'upload');
export const TEMP_DIR = path.resolve(__dirname, 'temp');
// 拆分chunk 这部分是应该在前端做的，这里是弄清楚分片的原理
export const splitChunks = async (filename: string, size: number = DEFAULT_SIZE) => {
    let filePath = path.resolve(__dirname, filename); // 要分割的文件绝对路径
    const chunksDir = path.resolve(TEMP_DIR, filename); // 以文件名命名的临时目录，存放分割后的文件
    await fs.mkdirp(chunksDir); // 递归创建目录
    let content = await fs.readFile(filePath);//Buffer 其实是一个字节数组 1个字节是8bit位
    let i = 0, current = 0, length = content.length;
    while (current < length) {
        await fs.writeFile(
            path.resolve(chunksDir, filename + '-' + i),
            content.slice(current, current + size)
        )
        i++;
        current += size;
    }
}
// splitChunks('header.jpg');
// 管道流
const pipeStream = (filePath:string, ws:WriteStream) => new Promise((resolve, _reject) => {
  const rs = fs.createReadStream(filePath);
  rs.on('end', async () => {
    await fs.unlink(filePath)
    resolve()
  })
  rs.pipe(ws)
})
// 合并chunk
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
// mergeChunks('header.jpg');

```

