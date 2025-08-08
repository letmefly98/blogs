# 大文件分片上传前后端开发

## 前端开发

### 分片

分片其实就是简单的 `File.slice` 操作，得到 `Blob` 对象数组，将其传给后端。  
后端再使用 `fs.writeFileSync(filePath, chunk)` 即可生成分片文件，后续再合并分片文件成原文件。

```ts
// 文件分片
const ChunkSize = 5 * 1024 * 1024 // 5MB
function getFileChunks(file: File, chunkSize = ChunkSize) {
  const result: Blob[] = [];
  let cur = 0;
  while (cur <= file.size) {
    const chunk = file.slice(cur, cur + chunkSize);
    result.push(chunk);
    cur += chunkSize;
  }
  return result;
}
```

### 上传

想传输 `Blob` 数据，只能使用 `FormData` 对象。   
因此 `headers` 也需改为 `multipart/form-data` 。   
此外上传普遍耗时较长，还需调整 `timeout` 入参。

```ts
// 对象转 FormData
function toFormData(params: Record<string, any>) {
  const formData = new FormData();
  Object.keys(params).forEach((key) => {
    const value = (params as any)[key];
    if (value !== undefined) formData.append(key, value);
  });
  return formData;
}
```

```ts
// 接口 - 上传文件
function ajaxUploadFile(params: Record<string, any>) {
  const formData = toFormData(params);
  return axios.post('https://', formData, {
    headers: { 'Content-Type': 'multipart/form-data' },
    timeout: 10 * 60 * 1000, // 10分钟
  })
}
```

### 并发池

如果一股脑地同时上传，可能会导致服务器负载过高，导致上传失败。如果一个接一个上传，则会导致上传速度过慢。  
因此可以使用并发池，按负载耐受进行并发请求。

```ts
// 并发池
interface Options {
  // 是否在错误时继续处理队列
  processWhenError?: boolean;
}
export class ConcurrencyPool {
  private queue: Array<() => void> = [];
  private running = 0;
  private limit: number = 3;
  private options: Options = {};

  constructor(limit: number, options?: Options) {
    this.limit = limit;
    this.options = { processWhenError: true, ...(options || {}) };
  }

  async run<T>(task: () => Promise<T>): Promise<T> {
    return new Promise((resolve, reject) => {
      const wrappedTask = async () => {
        this.running++;
        let hasError = false;
        try {
          const result = await task();
          resolve(result);
        } catch (error) {
          hasError = true;
          reject(error);
        } finally {
          this.running--;
          if (!hasError || (hasError && this.options.processWhenError)) this.processNext();
        }
      };
      if (this.running < this.limit) {
        wrappedTask();
      } else {
        this.queue.push(wrappedTask);
      }
    });
  }

  private processNext() {
    if (this.queue.length > 0 && this.running < this.limit) {
      const next = this.queue.shift()!;
      next();
    }
  }
}

```

```ts
// 业务 - 分片上传某某文件
interface HttpReqUploadFile {
  fileName: string
  file: Blob
  taskId: string
  chunkId: number
  chunkLength: number
}
async function uploadBizFile(file: File) {
  type Params = HttpReqUploadFile;
  const chunkList = getFileChunks(file)
  const chunkLength = chunkList.length
  const paramsList: Params[] = chunkList.map((chunk, index) => ({
    fileName: file.name,
    file: chunk,
    taskId: uuid(),
    chunkId: index,
    chunkLength,
  }))
  const pool = new ConcurrencyPool(5, { processWhenError: false })
  const uploadFileMethod = (p: Params) => pool.run(() => ajaxUploadFile(p))
  const result = await Promise.all(paramsList.map(uploadFileMethod))
  return result
}
```

### 断点续传

以上代码实现的，每次上传都会新建文件，如需避免这种情况，  
可将 `taskId` 改为文件的 `md5` 值，即可用以判断是否有相同文件。

另外，也可将 `chunkId` 改为文件的 `md5` 值，那么若本次上传中断，重新上传时可跳过已经上传过的分片。

## 后端开发

### 接收 FormData

```bash
npm i @koa/bodyparser @koa/multer
```

`@koa/bodyparser` 只接收普通 `json` 格式数据。用 `@koa/multer` 可实现接收 `FormData` 格式数据。

```ts
import { bodyParser } from '@koa/bodyparser';
import Router from '@koa/router';
import Koa from 'koa';
import type { Middleware } from 'koa';

const app = new Koa();
const router = new Router();

app.use(bodyParser({ formLimit: '500mb' }));

const apiUploadFile: Middleware = async (ctx, next) => {
  const props = ctx.request.body as HttpReqUploadFile;
  const file = ctx.request.file; // 注意：file 字段不在 body 中
}
router.post('/upload-file', uploader.single('file'), apiUploadFile);

app.use(router.routes());
app.use(router.allowedMethods());
app.listen(4002);
```

当然，你还可使用 `koa-body` 或 `koa-bodyparser` 等其他库来实现接收。

### 写入分片文件

需要注意两点，**分片与文件的关系**，以及 **分片的顺序** 。  
前者避免比如并发时同名文件的分片冲突，后者方便合并时按顺序读写。

我这边的方案是，将分片文件写入为 `/${taskId}/${chunkId}`，其中 `chunkId` 为索引。

```ts
import fs from 'fs';
import type { ParameterizedContext } from 'koa';
import path from 'path';

// 写入分片文件
function triggerChunkUpload(ctx: ParameterizedContext) {
  const props = ctx.request.body as HttpReqUploadFile;
  const file = ctx.request.file; // 即 chunk 对应 blob 封装的对象
  const chunkFilePath = path.join('uploaded', props.taskId, props.chunkId);
  console.log('写入分片文件', filePath);
  fs.writeFileSync(chunkFilePath, file.buffer);
}
```

### 合并分片文件

按顺序读取分片文件，再写入最终文件。  
读写毕竟是高 `IO` 操作，且最终文件可能较大，所以更推荐使用 `stream` 而非 `writeFile` 来实现读写。

```ts
// 合并分片文件
function mergeChunkUpload(ctx: ParameterizedContext) {
  return new Promise((resolve, reject) => {
    const props = ctx.request.body as HttpReqUploadFile;
    const chunkDir = path.join('uploaded', props.taskId);
    const resultFilePath = path.join('uploaded', props.fileName);
    const chunks = fs.readdirSync(chunkDir);

    const sorted = chunks.sort((a, b) => Number(a) - Number(b)).map((e) => path.join(chunkDir, e));
    const writeStream = fs.createWriteStream(filePath);

    sorted.forEach((chunk) => {
      const data = fs.readFileSync(chunk);
      writeStream.write(data);
    });
    writeStream.end();
    writeStream.on('finish', () => {
      console.log(`合并 ${sorted.length} 个分片文件到 ${filePath}`);
      const result = fs.existsSync(filePath);
      resolve(result);
    });
    writeStream.on('error', (err) => {
      reject(err);
    });
  });
}
```

所有分片写入完成后，可以由前端判断所有 `/chunk` 接口返回正确，然后调用 `/merge` 接口。  
也可后端根据 `chunkId` 和 `chunkLength` 自行判断分片已全部上传。

```ts
interface HttpResUploadFile {
  success: boolean;
  done: boolean;
}
const apiUploadFile: Middleware = async (ctx, next) => {
  const props = ctx.request.body as HttpReqUploadFile;
  let result: HttpResUploadFile = { success: false, done: false };

  // 写入分片
  await triggerChunkUpload(ctx);
  result = { success: true, done: false };

  // 分片全部上传完成，合并分片成文件
  const { chunkId = 0, chunkLength = 0 } = props;
  if (+chunkId === chunkLength - 1) {
    const resultFilePath = path.join('uploaded', props.fileName);
    await mergeChunkUpload(ctx);
    const ok = fs.existsSync(resultFilePath);
    result = { success: ok, done: true };
  }

  ctx.body = result;
  await next();
}
```
