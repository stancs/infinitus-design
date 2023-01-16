# Photo Sharing Service

## Requirements

1. Ability for the system to sync photos from remote sources (desktop, mobile, tablet, etc.)
2. Ability for a user to organize photos (one or more) into albums
3. Ability for a user to share photos or albums with one or more people
4. Ability for the system to understand images
   a. Faces
   b. Locations
   c. Common emotions and actions (happy, sad, crying, hugs, pitching, batting, wrestling)
5. Ability for a user to search photos based on understanding

# Clarifying Questions

## Questions for System Analysis
* Who is going to use it?
  - There will be three major types of clients: desktop, mobile phone, and tablet. A user will expect the photos in his devices will be synced so that the user can find photos in any of his devices

* How are they going to use it?
  - Share photos among the user's multiple devices
  - Organize photos in multiple albums
  - Pick multiple photos or albums and send a link to share with other people
  - Try to find some photos with a specific face or location
  - Try to find some photos with a specific emotion or actions


* What does the system do?
  - File synchronization services like DropBox or Google Drive do
  - Photo organization services like Apple Photo service does
  - Photo share service as Google drive does with a link that other people can download
  - Photo analyze service to extract features (face, location, emotions, actions)

* What are the inputs and outputs of the system?
  - Input: Photo from clients
  - Output: Metadata, Photo Management Data, Shared Link

## Key issues

- How to identify a file's uniqueness?
  <br>
  : The file's uniqueness will be determined by its key. If we choose a flat data model for object store as AWS S3, we create bucket and the bucket stores objects. There is no hierarchy of subbuckets or subfolders. However, we can infer logical hierarchy using key name prefixes and delimiters as below:

  `2012/summer/abc.jpg`

  `usa/california/sacramento/def.jpg`

  `summer-camping.jpg`

  In any device, two files with the same name won't exist. For a photo, its filename can be randomly generated or incrementally generated to keep its uniqueness within a certain folder or sublocation. So it won't be a problem to use a `key` that includes prefix(subfolder name) and demiliter(`/`) to identify a file uniquely.

- When we search photos, what if the explored photos are too many?
  <br>
  : We might use pagination to retrieve data rapidly in the backend. Due to the search conditions, we might have too much data after searching the database. In this case, the response overhead will be huge, and it will take quite much time to get the full search result. In this case, retrieving data in small chunks with a short waiting time is better. We can set the size of each pagination (such as 20 data sets) and repeatedly receive data from the backend per page.

- Based on client type(desktop, mobile, tablet, etc.), will we apply any difference?
  <br>
  : The three types of clients, as mentioned earlier, might have different file structures internally. As we can see from Mac or Windows systems, the desktop usually has a general subfolder structure. Tablets and mobile phones might have different file structures with photos in special locations. To sync photos among those three kinds of clients, we need to create a specially named folder on a desktop that matches with photo files in tablet/mobile phones. But I don't want to go into much more detail since this over-complicates this system design problem.

- How to resolve sync conflicts?
  <br>
  : When there are conflicts when several clients try to update their file to the backend server, we can set the first one that gets processed as the winner, but the latter one gets a conflict. In this case, the latter one gets a conflict error, and it needs to merge the latest change and try to update again.

- Do we need to split the photo files into several blocks?
  <br>
  : For 12 Megapixel picture in iPhone HEIC format, the file size varies between 1.5 MB and 3.5 MB. However, the photo size can rapidly increase as we use a different camera with a higher resolution. It's better to split a file with several fixed size of chunks. (Let's call this chunk a block) Block-based file management has several advantages. For example, in a mobile environment with slow network speed, it would take a long time to upload a file of 10MB. Whenever a failure happens, we must re-upload the entire file to ensure uncorrupted files are in the backend server. However, if we split them into several blocks, we can sequentially upload each block to the server. If a failure happens, we can resume the process from the failed block. This procedure can save a significant amount of time in synchronization. 
  
  However, in this system design problem, I decided not to add this block-based file management approach. We can assume that most digital photos are less than 25MB, significantly less than GB-sized files. For only photo files, it's okay not to split them into several chunks. But as mentioned above, there is a benefit for the split and managing blocks of files. 

# Features

## Functional requirements

- Sync photos across multiple devices (add photos, download photos)
- Organize photos into albums
- Share photos or albums with one or more people
- Analyze images to extract info about faces, locations, and common emotions and actions
- Search photos based on analyzed understanding (faces, locations, emotions, and actions)

## Non-functional requirements

- High availability
- Scalability
- Reliability
- Fast sync speed
- Bandwidth usage

## High Level Design

![Highlevel Design](/diagram/highlevel_design.png)

https://lucid.app/lucidchart/8e676fb9-6266-495e-9484-70d411d05049/edit?viewport_loc=-328%2C-473%2C2219%2C2515%2C0_0&invitationId=inv_2fe7733f-ff5f-415a-affe-dc833a0facf4

# Component Design

## Client Design

![Client Design](/diagram/client_design.png)

- Client Photo Metadata Database: This lightweight database running on the client side will store information about different files in the workspace using file versions.

- Monitor: Monitor constantly checks for file changes in the workspace. If any of the files/folders in the workspace are created, updated, added, or removed, then it will notify Indexer about the changes

- Indexer: Indexer listens for the events from the Monitor and updates Client Photo Metadata Database about the updated photos. After committing file changes to Client Database, this also notifies Synchronizer.

- Synchronizer: Synchronizer listens for the events from Indexer and Notification Service and communicates with Meta Service and Cloud Object Storage to sync photo files in the workspace.

## Backend Design

![Backend Design](/diagram/backend_design.png)
- Meta Service: Meta Service is responsible for synchronizing the file metadata between the client and backend server. Suppose there is any discrepancy between the current client and the backend server. In that case, Meta Service will create pre-signed URLs for uploading and downloading photos after communicating with a cloud object store like AWS S3. This service also notifies any file changes for different clients to Notification Service.

- Photo Management Service: Photo Management Service is responsible for organizing photos into albums and creating a link for shared photos or albums with multiple people.

- Photo Analyzer Service: Photo Analyzer Service will scan the contents of the user's photos and extracts information about faces, locations, common emotions, and actions (happy, sad, crying, hugs, pitching, batting, wrestling). Whenever any metadata is created or updated, the metadata will be sent to a message queue to be handled sequentially in Photo Analyzer Service server. Photo Analyzer Server will check newly created/updated photo files from Cloud Object Store and analyze them to extract info. The extracted features will be stored in Metadata Database and Metadata Cache.

- Notification Service: Notification Service broadcasts all the file changes for a specific user to all connected clients to ensure all the files are synced up across multiple clients.

- User Service (optional): User Service handles all the user information, such as account information. Recognized face information can use this information to match analyzed face information

# Design Deep Dive

## File Sync
File sync will be performed:

[Normal Updates] 
: On the client side, the Synchronizer will listen for the events from the Indexer for any local file changes and the Notification service for remote changes. If there are any changes, the client will communicate with Meta Service to get the latest versions of the file data. 
In this case, uploading & downloading files will be performed since we have a list of files to update from the Indexer and the Notification service.

If the local files is the newest file, then the Synchronizer will update the file metadata in the backend with higher version, but set the status as 'PENDING' since the client needs to update to Cloud Object Store. After finishing updating the file to Cloud Object Store, it will change the file metadata status as "COMPLETE"

[Initial Update]
: If there is nothing in the client workspace, the Synchronizer can check local files in a specific folder. It will run BST (Breadth First Search) for local files in a specific folder. 
![File BST](/diagram/file_bst.png)

Then, the client can contact Metadata service to get the remote metadata to compare both file lists. 
Each file or folder has `version` to see which is the latest. After checking the latest versions of files, the client will get the pre-signed download URLs from Meta service and directly download the files from Cloud Object Store. 

## File Share
After a user pick specific photo files or specific album, they can share these photos or album with other people using a specific link. 

- When a user picks several photos or an album, the client communicates with the Photo Management service to create a link to share those photos or albums.
- Photo Management service will create a link in API Gateway so that the user can access the photo with a simple link
- Photo Management service will create pre-signed links for photos so that any user can download them without any issue with authentication or authorization
- API Gateway is connected to Photo Management service. When a user connects to this link, they can see the list of photos or albums in HTML/CSS (server-rendered) so that they can download them easily

## Photo Analyzer
Whenever a photo is added or updated, its metadata will be entered into a message queue for Photo Anayzer to ensure better performance and increased reliability. When the message reaches to Photo Analyzer, the service will download each photo to extract features such as face info, location, emotions, and actions. The analyzed result will be stored in the Metadata database and caches

## APIs

The clients can use both GraphQL API and REST API. In my design, I prefer to use GraphQL for clients to communicate with Meta Service and Photo Mgmt Service. But in actual downloading and uploading photo files in client devices, I prefer to use REST API for clients to directly connect to the Cloud Object Store to reduce the load to the backend server.

## REST API or GraphQL API?

### Benefits of GraphQL:

- Data fetching control
  : data request/response customization
- Using multiple data sources: deliver more data
  : deliver more data to the client for a single request
- Rapid prototyping
  : exposes a single endpoint to allow the clients to access multiple resources
- Single endpoint
  : GraphQL provides one single endpoint, which provides a simple connection point to users

### Disadvantage of GraphQL:

- Needs carefully designed schema
- Performance issue
  : Multiple network calls can delay response time

### Why use GraphQL over REST?

- Single endpoint for plenty of file sync up
- Multiple types of clients (phone, tablet, desktop, etc.): Data fetching customization (request/response) is possible with GraphQL to pick right amount of data for each client
- Not complicated GraphQL schema

## GraphQL API designs

### Get Sync List

In a specific location (prefix), retrieve the current file metadata (key, version, etc.) from the backend server so that client synchronizer can compare them with its current file list

```
type Query {
  syncList(prefix: String!): [FileMetadata!]
}

type FileMetadata {
  key: String!
  version: String!
}
```

### Create Presigned Upload URLs

[Schema]

```
type Mutation {
  createPresignedUploadURLs(files: [UploadFileInfo!]!): UploadURLResponse!
}

type UploadFileInfo {
  key: String!
}

type FileUploadURL {
  key: String!
  uploadURL: String!
}

type UploadURLResponse {
  success: Boolean!
  msg: String
  uploadURLs: [FileUploadURL!]!
}
```

[Examples]
header: {
...
userId,
...
}

```
mutation createPresignedUploadURLs(input: [
    {
      key: "2012-summer/abc.jpg"
    }
  ]) {
  success
  msg
  uploadURLs: {
    key
    uploadURL
  }
}
```

- In server side, the server will retrieve the first prefix name from user ID in the request header (JWT)
- There will be a pre-determined values for server side and will be used to create presigned URL
  (ex. timeoutDuration, S3 region, etc.)
  https://dev.to/franciscomendes10866/upload-files-to-s3-object-storage-or-minio-with-apollo-server-4m46

### Create Presigned Download URLs

[Schema]

```
type Mutation {
  createPresignedDownloadURLs(files: [DownloadFileInfo!]!): DownloadURLResponse!
}

type DownloadFileInfo {
  key: String!
}

type FileDownloadURL {
  key: String!
  downloadURL: String!
}

type DownloadURLResponse {
  success: Boolean!
  msg: String
  downloadURLs: [FileDownloadURL!]!
}
```

[Examples]
header: {
...
userId,
...
}

```
mutation createDownloadURLs(input: [
    {
      key: "2012-summer/abc.jpg"
    }
  ]) {
  success
  msg
  downloadURLs: {
    key
    downloadURL
  }
}
```

### Create album

```
type Mutation {
  createAlbum(fileName: String!): AlbumMutationResponse
}

type AlbumMutationResponse {
  albumId: ID!
  success: Boolean!
  msg: String
}
```

```
mutation createAlbum(name: "Friends") {
  albumId
  success
  msg
}
```

(We might need `updateAlbum(...)` and `removeAlbum(...)` mutations for more detailed control of albums. But we will skip those for now to simplify this design)

### Add photos to an album

```
type Mutation {
  addPhotosToAlbum(fileNames: [String!]!, albumId: ID!): [PhotoMutationResponse!]!
}
type PhotoMutationResponse {
  fileName: String!
  success: Boolean!
  msg: String
}
```

```
mutation addPhotosToAlbum(fileNames: ["abc.jpg", "def.jpg"], albumId: "ALBUM001") {
  fileName
  success
  msg
}
```

### Create photo share URL

```
type Mutation {
  createPhotoShareURLs(files: [FileInfo!]!): UploadURLResponse!
}

type FileInfo {
  key: String!
}

type FileUploadURL {
  key: String!
  uploadURL: String!
}

type UploadURLResponse {
  success: Boolean!
  msg: String
  uploadURLs: [FileUploadURL!]!
}
```

### Get Photo Metadata

```
type Query {
  getPhotoMetadata(key: String!): FileMetadata
}

type FileMetadata {
  key: String
  kind: FILE_OR_FOLDER
  version: String
  status: String
  downloadURL: String
  albumIds: [String]
  faceIds: [ID]
  locationsIds: [ID]
  emotionIds: [ID]
  actionIds: [ID]
}
```

### Update Photo Metadata

```
type Mutation {
  updatePhotoMetadata(key: String!, version: String!, status: String!): FileMetadata
}

type FileMetadata {
  key: String
  kind: FILE_OR_FOLDER
  version: String
  status: String
  downloadURL: String
  albumIds: [String]
  faceIds: [ID]
  locationsIds: [ID]
  emotionIds: [ID]
  actionIds: [ID]
}
```

### Search Photo

```
type Mutation {
  searchPhoto(face: FaceSearchItems, location: LocationItems, emotion: EmotionItems, action: ActionItems): [FileMetadata]
}

type FileMetadata {
  key: String
  kind: FILE_OR_FOLDER
  version: String
  downloadURL: String
  albumIds: [String]
  faceIds: [ID]
  locationsIds: [ID]
  emotionIds: [ID]
  actionIds: [ID]
}
```

# Database Design

## DB Choices (SQL vs NoSQL)

Prefer to use SQL Database due to:
  * Need to store lots of amounts of data
  * Easy to scale using data sharding
  * Simple data structure 
  * No need for complex join
  * Non-relational data
  

[Ref] Tradeoff between SQL and NoSQL: 
  * Reasons for SQL:
    * Structured data
    * Strict schema
    * Relational data
    * Need for complex join
    * Transactions
    * Lookups by index are very fast
  * Reasons for NoSQL:
    * Semi-structured data
    * Dynamic or flexible schema
    * Non-relational data
    * No need for complex joins
    * Store many TB (or PB) of data
    * Very data intensive workload

## Partition Key and Sort Key to find right file(s)

For example, if we use AWS DynamoDB, we can use `composite primary key` to search files. This type of key is composed of two attributes, `partition key` and `sort key`. For file key search, we can assign as below:

- paritition key: folder name except the file name
  (ex: `/`, `/2022-summer`)
- sort key: file name
  (ex: `sanjose-01022023.jpg`)

For example, if we want to store metadata of `/2022-summer/sanjose-01022023.jpg` file into the database, the partition key will be `/2022-summer` and sort key will be `sanjose-01022023.jpg`. 

The reason why we split the full file location string into folder string and filename string is for the convenience to find files in a specific location. If we want to search for all the files in `/2022-summer` folder, we can search all the files with this partition key. 

However, if we want to find a specific file, then we can search using the composition key (partition key + sort key) to find a particular file.

Reference: https://aws.amazon.com/blogs/database/choosing-the-right-dynamodb-partition-key/

## DB Schema

### File

```
{
  "folder": "/summer-2022",
  "file": "sanjose-01022023.jpg",
  "status": "PENDING",
  "kind": "FILE"
  "uploadURL": "",
  "uploadURLExpiration": "",
  "downloadURL": "",
  "downloadURLExpiration": "",
  "albumIds": ["ALBUM_001", "ALBUM_003"],
  "faceIds": ["FACE_ABC"],
  "emotionIds": ["EM_HAPPY"],
  "actionIds": ["ACT_BATTING"],
}
```

### Share Link
```
{
  "shareLink": "http://sharelink.com/abcabc",
  "expiration": "2023-02-01T14:48:00.000Z",
  "html": "<http> ... </html>"
}
```

# Further Discussion Issues

- Data sharding
  : One of the major reason that I choose NoSQL database is for scalability since I expect the amount of file metadata will be significant. We can realize horizontal scaling by spliting a single dataset into multiple partitions or shards. A single machine can store and process only a limited amount of data, but this database sharding overcomes this limitation by splitting data into smaller chunks and storing them across several database servers. 

- How to de-duplicate
  : There is only one file in a specific folder with a specific filename for a specific user. So key `/2022-summer/1001.jpg` file will exist uniquely. If there is different versions of files across several devices, we can compare versions and sync up again.

- Multiple photo upload issues
  : We might limit the number of photos for uploading to a certain number. For example, we can set 10 photos as a concurrent photo upload limit. In this case, we can sequentially upload each photo

# Future works

- Block-based file sync
