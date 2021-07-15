# nft.storage

## Steps for usage

-Create an account https://nft.storage/login/
-Create and backup token https://nft.storage/manage/
-Select the upload method applicable to the front end (below)
  Note: The generated hash value and path cannot be directly accessed, and front-end splicing is required

```javascript
import {NFTStorage} from "nft.storage"
const client = new NFTStorage({
  token: <API_TOKEN>
})
```

-View uploaded files https://nft.storage/files/
-Code View (Front End)

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

## upload files

-Upload syntax (and get metadata)

```javascript

import {File} from'nft.storage'

const metadata = await client.store({
  name:'<NAME>',
  description:'<DESCRIPE_FILE>',
  image: new File([<DATA>],'<FILE_NAME>', {type:'image/jpg' })
})
console.log(metadata)

```

hash value: metadata.ipnft

-Access method (Front-end applicable 🌟🌟🌟)
  https://<HASH>.ipfs.dweb.link/<FILE_NAME>

-Disadvantages
  Upload content restriction string
  Front-end splicing two variables

(image)

## Upload folder

-Upload syntax (and get hash directly)

```javascript

import {File} from'nft.storage'

const hash = await client.storeDirectory([
  new File([<DATA>],'<FILE_NAME>'),
  new File([JSON.stringify(<DATA>)],'<FILE_NAME>')
])

```

hash value: hash

-Access method (Front-end applicable 🌟🌟🌟)
  https://<HASH>.ipfs.dweb.link/<FILE_NAME>

-Disadvantages
  Upload content restriction string
  Front-end splicing two variables
  