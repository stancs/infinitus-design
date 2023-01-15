# Photo Sharing Service

## Requirements

1. Ability for the system to sync photos from remote sources (desktop, mobile, tablet, etc)
2. Ability for a user to organize photoes (one or more) into albums
3. Ability for a user to share photos or albums with one or more people
4. Ability for the system to understand images
   a. Faces
   b. Locations
   c. Common emotions and actions (happy, sad, crying, hugs, ptching, batting, westling)
5. Ability for a user to search photos based on understanding

# Clarifying Questions

Who is going to use it?
How are they going to use it?
How many users are there?
What does the system do?
What are the inputs and outputs of the system?
How much data do we expect to handle?
How many requests per second do we expect?
What is the expected read to write ratio?

## Key issues

- How to identify file's uniqueness?
  <br>
  : The file's uniqueness will be determined by its key. If we choose a flat data model for object store as AWS S3, we create bucket and the bucket stores objects. There is no hierarchy of subbuckets or subfolders. However, we can infer logical hierarchy using key name prefixes and delimiters as below:

  `2012/summer/abc.jpg`

  `usa/california/sacramento/def.jpg`

  `summer-camping.jpg`

  In any device, two files with the same name won't exist. For photo, it's filename can be randomly generated or incrementally generated to keep its uniqueness within a certain folder or sublocation. So it won't be a problem to use a `key` that includs prefix(subfolder name) and demiliter(`/`) to uniquely identify a file

- When we search photos, what if the searched photos are too many?
  <br>
  : We might use pagination to retrieve data rapidly in the backend. Due to the search conditions, we might have too many data after searching database. In this case, the response overhead will be huge and it will take quite much time to get the entire search result. In this case, it's better to retrieve data in small chunks with low waiting time. We can set the size of each pagination (such as 20 data sets) and repeatedly receive data from the backend per page.

- Based on client type(desktop, mobile, tablet, etc.), will we apply any difference?
  <br>
  : The above mentioned three types of clients might have different file structure internally. Desktop might have general subfolder structure as we can see from Mac or Windows systems. Tablet and mobile phone might have a different file structure that comes with photos in special locations. To sync photos among those three kinds of clients, we might need to create a specially named folder in a desktop that matches with photo files in tablet/mobile phones. But I don't want to go into much more details since this over-complicate this system design problem.
- How to resolve sync conflicts?
  <br>
  : When there are conflicts when several clients try to update their file to the backend server, we can set the first one that gets process wins but the later one gets a conflict. In this case the later one gets a conflict error, and it needs to merge the latest change and try to update again.
- Do we need to split the photo files into several blocks?
  <br>
  : For 12 Megapixel picture, in iPhone HEIC format, the file size varies between 1.5 MB and 3.5 MB. However the photo size can increase rapidly as we use different camera with higher resolution. I believe it's better to split a file with several fixed size of chunks. (Let's call this chunk as a block) Block-based file management has several advantages. For example, in a mobile environmwent with slow network speed, it would take a long time to upload a file of 10MB. Whenever a failure happens we need to re-upload the entire file to make sure uncorruped files in the backend server. However, if we split them into several blocks, we can sequentially upload each block to the server. If a failure happens, we can resume the process from the failed block. This procedure can save a significant amounts of time for synchronization

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

- Monitor
- Indexer
- Client Photo Metadata Database
- Synchronizer

## Backend Design

- Meta Service
- Photo Analyzer Service
- Notification Service
- User Service (optional)

# API

## REST API or GraphQL API?

### Benefits of GraphQL:

- Data fetching control
  : data request/response customization
- Using multiple data sources: deliver more data
  :deliver more data to the client for a single request
- Rapid prototyping
  : exposes a single end point to allow the clients to access multiple resources

### Disadvantage of GraphQL:

- Needs carefully designed schema
- Performance issue
  : Multiple network calls can delay response time

# API designs

## Get Sync List

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

## Create Upload URLs

[Schema]

```
type Mutation {
  createUploadURLs(files: [UploadFileInfo!]!): UploadURLResponse!
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
mutation createUploadURLs(input: [
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

- In server side, the server will retrieve bucket name from user ID in the request header (JWT)
- There will be a pre-determined values for server side and will be used to create presigned URL
  (ex. timeoutDuration, S3 region, etc.)
  https://dev.to/franciscomendes10866/upload-files-to-s3-object-storage-or-minio-with-apollo-server-4m46

## Create Download URLs

[Schema]

```
type Mutation {
  createDownloadURLs(files: [DownloadFileInfo!]!): DownloadURLResponse!
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

<!-- ## Sync photos

Set up some syncId to compare different photos in the backend. And get need-to-be-updated photo lists from the backend
After everything is done, update syncId to the latest one

https://docs.aws.amazon.com/AmazonS3/latest/userguide/download-objects.html -->

## Create album

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

## Add photos to an album

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

## Get all faces

## Create photo share URL

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

## Get Photo Metadata

```
type Query {
  getPhotoMetadata()
}
```

## Search Photo

```
type Mutation {
  searchPhoto(face: FaceSearchItems, location: LocationItems, emotion: EmotionItems, action: ActionItems) {

  }
}
```

- Upload to object store directly
  (https://aws.amazon.com/blogs/compute/uploading-to-amazon-s3-directly-from-a-web-or-mobile-application/)
  PUT <presigned_URL>
  body: {
  <fileBinary>
  }

Issues

- Data sharding
- How to de-duplicate
- Multiple photos upload issues
  : We might limit the number of photos for uploading to a certain number. For example, we can set 10 photos as concurrent photo upload limit. In this case, we can sequentially upload each photo
