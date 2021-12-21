# Nebula Drive


![Build Status](https://github.com/Telios-org/nebula-drive/actions/workflows/node.js.yml/badge.svg)

Nebula drives are real-time distributed storage for files and key value databases built on top of [Hypercore Protocol](https://hypercore-protocol.org/). This project exists because the [Telios](https://telios.io) email client needed a way to distribute and store encrypted emails on user's local file systems over a peer-to-peer (P2P) network. A lot of inspiration was taken from [Hyperdrive](https://github.com/hypercore-protocol/hyperdrive), but Hyperdrive didn't have options for fine-grain access control, multiple writers, and the ability to delete files from disk once added to the drives.

Nebula drives come with a handful of useful features like:
- __Shareable over company firewalls and mobile networks__: The P2P network runs on [Hyperswarm](https://github.com/hyperswarm/hyperswarm) which has the ability to hole-punch through most company firewalls and mobile connections.
- __Full Text Search__: Create encrypted full text search indexes on top of Hypercores.
- __Access Control__: Control access to each file by sharing the file's hash and the drive's discovery key.
- __Multiwriter__: Drives can have multiple peers with write access by exchanging eachother's keys.
- __Collections__: Along with files, drives can create and share simple key value btree databases built on [Hyperbee](https://github.com/hypercore-protocol/hyperbee). Collections also have the option to be encrypted with a secret key.

### TODOs:
- [x] Connect to drives behind corporate firewalls and mobile networks
- [x] Create and share key value databases between peers
- [x] Upgrade multiwriter to Hypercore v10
- [x] Build full text search indexes from encrypted Hypercores
- [ ] Upgrade access control to limit sharing by a peer's public key
- [ ] Turn an existing directory into a drive and watch for changes
- [ ] Upgrade collections API with [Hyperbeedeebee](https://github.com/RangerMauve/hyperbeedeebee)

## Installation

```js
npm i @telios/nebula-drive
```

## Usage

```js
/******************************************************
 * 
 * Create a new drive and write encrypted files to it
 * 
 ******************************************************/

const Drive = require('nebula-drive')

// Optionally pass in an encryption key to encrypt the drive's databases
const encryptionKey = Buffer.alloc(32, 'hello world')

const localDrive = new Drive(__dirname + "/drive", null, { 
  keyPair,
  encryptionKey,
  swarmOpts: {
    server: true,
    client: true
  }
})

await localDrive.ready()

// Key to be shared with other devices or services that want to seed this drive
const drivePubKey = localDrive.publicKey

// Clone a remote drive
const remoteDrive = new Drive(__dirname + "/drive_remote", drivePubKey, { 
  keyPair,
  swarmOpts: {
    server: true,
    client: true
  }
})

await remoteDrive.ready()

localDrive.on('file-sync', file => {
  // Local drive has synced somefile.json from remote drive
})

// Write a non-encrypted file to the drive
await remoteDrive.writeFile('/dest/path/on/drive/somefile.json', readableStream)

// Write anencrypted file to the drive
await remoteDrive.writeFile('/dest/path/on/drive/someEncryptedFile.json', readableStream, { encrypted: true })


/******************************************************************
 * 
 * Create an encrypted and shared database with full text search
 * 
 *****************************************************************/

const corpus = [
  {
    id: 'p1',
    title: 'Painting 1',
    text_body: "In your world you can create anything you desire."
  },
  {
    id: 'p2',
    title: 'Painting 2',
    text_body: "I thought today we would make a happy little stream that's just running through the woods here."
  },
  {
    id: 'p3',
    title: 'Painting 3',
    text_body: "See. We take the corner of the brush and let it play back-and-forth. No pressure. Just relax and watch it happen."
  },
  {
    id: 'p4',
    title: 'Painting 4',
    text_body: "Just go back and put one little more happy tree in there. Without washing the brush, I'm gonna go right into some Van Dyke Brown."
  },
  {
    id: 'p5',
    title: 'Painting 5',
    text_body: "Trees get lonely too, so we'll give him a little friend. If what you're doing doesn't make you happy - you're doing the wrong thing."
  },
  {
    id: 'p6',
    title: 'Painting 6',
    text_body: "Son of a gun. We're not trying to teach you a thing to copy. We're just here to teach you a technique, then let you loose into the world."
  }
]

const collection = await drive.db.collection('BobRoss')

for(const data of corpus) {
  await collection.put(data.id, { title: data.title, text_body: data.text_body })
}

const doc = await collection.get('p4')

// doc
// {
//   key: 'p4'
//   value: {
//     title: 'Painting 4',
//     text_body: "Just go back and put one little more happy tree in there. Without washing the brush, I'm gonna go right into some Van Dyke Brown."
//   }
// }

await collection.ftsIndex(['title', 'text_body'])

const query = await collection.search("happy tree")
```

## API / Examples

#### `const drive = new Drive(storagePath, [key], [options])`

Create a drive to be shared over the network which can be replicated and seeded by other peers.

- `storagePath`: The directory where you want the drive to be created.
- `key`: The public key of the remote drive you want to clone

Options include:

```js
{
  encryptionKey,  // optionally pass an encryption key to encrypt the drive's database
  keyPair: { // ed25519 keypair
    publicKey, 
    secretKey
  },
  swarmOpts: { // Set server to true to start this drive as a server and announce its public key to the network
    server: true | false,
    client: true | false
  }
}
```

```js
const Drive = require('nebula-drive')

// Create a new local drive.
const localDrive = new Drive(__dirname + "/drive", null, { 
  keyPair,
  swarmOpts: {
    server: true,
    client: true
  }
})

await localDrive.ready()

// Key to be shared with other devices or services that want to seed this drive
const drivePubKey = localDrive.publicKey

// Clone a remote drive
const remoteDrive = new Drive(__dirname + "/drive_remote", drivePubKey, { 
  keyPair,
  swarmOpts: {
    server: true,
    client: true
  }
})

await remoteDrive.ready()
```

#### `await drive.ready()`

Initialize the drive and all resources needed.

#### `await drive.addPeer(publicKey)`

Adds a remote drive as a new writer. After a peer has been added, the drive will automatically try to reconnect to this peer after every restart.

Example Usage:

```js
// Local drive on Device A
const drive1 = new Drive(__dirname + "/drive", null, { 
  keyPair,
  swarmOpts: {
    server: true,
    client: true
  }
})

// Local drive on Device B
const drive2 = new Drive(__dirname + "/drive", null, { 
  keyPair,
  swarmOpts: {
    server: true,
    client: true
  }
})


await drive2.addPeer(drive1.publicKey)
```

#### `await drive.removePeer(publicKey)`

Stop replicating with another drive peer.


#### `const file = await drive.writeFile(path, readableStream, [opts])`

Write a file from a readable stream. When choosing to encrypt a file, the encryption key will be passed back in the response. Each file is encrypted with a unique key which should be stored separately.

- `path`: Full path where the file resides on the local drive `dir/to/my/file.jpg`
- `readableStream`: Any readableStream `fs.createReadableStream()`

Options include:
```js
// When encrypted is true a key and header value will be returned after the file has been written
{ 
  encrypted: true 
}
```

#### `const stream = await drive.readFile(path)`

Creates a readable stream of data from the requested file path.

- `path`: Full path where the file resides on the local drive `dir/to/my/file.jpg`

<!-- #### `const readableStream = await drive.fetchFileByHash(fileHash)`

Similar to how IPFS uses (content addressing)[https://proto.school/content-addressing/03], a drive can fetch a file by the hash of it's contents. If another drive is announcing the file hash you're looking for then a readable stream will be returned.

- `fileHash`: Hash of the file's contents. -->

#### `const stream = await drive.fetchFileByDriveHash(discoveryKey, fileHash, [opts])`

Drives with many files may not want to announce every file by it's hash due to network bandwidth limits. In this case, a drive has the option of sharing it's `discoveryKey` which peers can use to connect to the drive and then make a request file hash request.

- `discoveryKey`: Remote drive's discovery key `drive.discoveryKey` which is used by peers to request resources from the drive.
- `fileHash`: Hash of the file being requested on the remote drive.
- `opts`: If a key and header are passed in then the return stream will be the deciphered data
  - `key`: Encryption key used for deciphering the encrypted stream. This key is returned from the `drive.writeFile` method.
  - `header`: Needed for validating the encrypted stream. This gets returned from `drive.writeFile()`.

#### `const stream = drive.decryptFileStream(stream, key, header)`

If `drive.fetchFileByDriveHash` is returning encrypted data, then `decryptFileStream` will transform that stream and return a new stream of deciphered data.

- `stream`: Readable stream of encrypted data
- `key`: Encryption key used for deciphering the encrypted stream. This key is returned from the `drive.writeFile` method.
- `header`: Needed for validating the encrypted stream. This gets returned from `drive.writeFile()`.

#### `await drive.fetchFileBatch(files, cb)`

Fetching files as a batch automatically chunks parallel requests in a fixed batch size so a drive can request as many files as it needs without impacting performance.

- `files`: Array of file objects with the following structure
  - `discovery_key`:  Remote drive's discovery key `drive.discoveryKey` which is used by peers to request resources from the drive.
  - `hash`: Hash of the file being requested on the remote drive.
  - `key`: Encryption key used for deciphering the encrypted stream. This key is returned from the `drive.writeFile` method.
  - `header`: Needed for validating the encrypted stream. This gets returned from `drive.writeFile()`.
- `cb`: Callback method that runs after every file stream has been initialized. Use this for handling what to do with the individual file streams. Note that this should return a promise.

Example Usage:

```js

await drive.fetchFileBatch(files, (stream, file) => {
  return new Promise((resolve, reject) => {
    const writeStream = fs.createWriteStream(`./${file.path}`)
    pump(stream, writeStream, (err) => {
      resolve()
    })
  })
})

```

#### `await drive.close()`

Fully close the drive and all of it's resources.

#### `drive.on('message', (peerPubKey, socket) => {})`

Emitted when the drive has recieved a message from a peer.

- `peerPubKey`: Public key of the peer that sent the message
- `socket`: The socket returned on this event can be used as a duplex stream for bi-directional communication with the connecting peer. `socket.write` `socket.on('data, data => {})`

#### `drive.on('file-add', (file, enc) => {})`

Emitted when a new file has been added to a local drive.

- `file`: A file object
  - `path`: drive path the file was saved to
  - `hash`: Hash of the file
- `enc`: Passes back properties needed to decrypt the file
  - `key`: Key needed to decrypt the file
  - `header`: Needed for validating the encrypted stream

#### `drive.on('sync', () => {})` 

Emitted when the drive has synced any remote data.

#### `drive.on('file-sync', (file) => {})`

Emitted when the drive has synced remote a remote file.

#### `drive.on('file-unlink', (file) => {})`

Emitted when a file has been deleted on the drive.

#### `drive.on('fetch-error', (err) => {})`

Emitted when there has been an error downloading from the remote drive

#### `drive.on('network-updated', (network) => {})`

Emitted when either the internet connection or the drive's connection to Hyperswarm has changed

Returns:
- `network`
  - `internet`: true|false
  - `drive`: true|false

## Drive Database API

#### `const collection = await drive.db.collection(name)`

Creates a new key value collection. Collections are encrypted with the drive's `encryptionKey` (`drive.encryptionKey`) when the key is passed in during initialization.

#### `await collection.put(key, value)`

Inserts a new document into the collection. Value should be a JSON object.

#### `await collection.get(key)`

Get a document by it's key

#### `await collection.del(key)`

Deletes a document by it's key

#### `await collection.ftsIndex([prop1, prop2, ...])`

Create a full text search index from a collection's properties

```js
await collection.ftsIndex(['address', 'first_name', 'last_name'])
```

#### `const results = await collection.search(query, [opts])`

Query a searchable index

Options include:
```js
{ 
  limit: 10 
}
```