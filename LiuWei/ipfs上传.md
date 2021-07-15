# nft.storage

## 使用步骤

- 创建账户 https://nft.storage/login/
- 创建并备份 token https://nft.storage/manage/
- 选择前端适用的上传方式（下述）
  注：生成的 hash 值和路径不能直接访问，需前端拼接

```javascript
import { NFTStorage } from "nft.storage"
const client = new NFTStorage({
  token: <API_TOKEN>
})
```

- 查看上传的文件 https://nft.storage/files/
- 代码查看（前端）

```javascript
axios
  .get(`https://api.nft.storage`, {
    headers: {
      Authorization: "Bearer <TOKEN>",
      platform: "web",
    },
  })
  .then((data) => {
    console.log(JSON.parse(data.request.response));
  });
```

（图片）

## 上传文件

- 上传语法（并获取 metadata）

```javascript

import { File } from 'nft.storage'

const metadata = await client.store({
  name: '<NAME>',
  description: '<DESCRIPE_FILE>',
  image: new File([<DATA>], '<FILE_NAME>', { type: 'image/jpg' })
})
console.log(metadata)

```

hash 值：metadata.ipnft

- 访问方式（前端适用 🌟🌟🌟）
  https://<HASH>.ipfs.dweb.link/<FILE_NAME>

- 弊端
  上传内容限制字符串
  前端拼接两个变量

（图片）

## 上传文件夹

- 上传语法（并直接获取 hash）

```javascript

import { File } from 'nft.storage'

const hash = await client.storeDirectory([
  new File([<DATA>], '<FILE_NAME>'),
  new File([JSON.stringify(<DATA>)], '<FILE_NAME>')
])

```

hash 值：hash

- 访问方式（前端适用 🌟🌟🌟）
  https://<HASH>.ipfs.dweb.link/<FILE_NAME>

- 弊端
  上传内容限制字符串
  前端拼接两个变量

## 上传 buffer 文件

- 上传语法（并直接获取 hash）
  const hash = await client.storeBlob(<BLOB || BUFFER>)

- 访问方式（前端适用 🌟🌟🌟🌟🌟）
  https://<HASH>.ipfs.dweb.link/

# puppeteer

**生成页面的屏幕截图和 PDF。**

```javascript
const puppeteer = require("puppeteer");

(async () => {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();
  await page.setRequestInterception(true);
  page.on("request", (request) => {
    if (request.resourceType() === "image") request.abort();
    else request.continue();
  });
  await page.goto("<URL>");
  await page.screenshot({ path: "<FILE_NAME>.png", fullPage: true });
  await page.pdf({
    path: "<FILE_NAME>.pdf",
  });
  await browser.close();
})();
```
