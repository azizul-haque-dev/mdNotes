# 12. File Upload — NestJS

ধরুন আপনি একটা **e-commerce application** বানাচ্ছেন। User profile picture upload করবে, farmer product image upload করবে।

Browser থেকে file আসে এভাবে:

```text
Browser
   │
   │ image.jpg
   ▼
NestJS Server
   │
   ├── Validate file
   ├── Check MIME type
   ├── Check file size
   │
   ▼
Storage
   ├── Local disk
   ├── Cloudinary
   └── Cloudflare R2
```

এখন একে একে বুঝি।

---

# 1. Multer

## Multer কী?

**Multer হলো Node.js/NestJS application-এ uploaded file receive করার middleware।**

সাধারণ JSON request:

```http
POST /users
Content-Type: application/json
```

এখানে data হয়:

```json
{
  "name": "Aziz",
  "email": "aziz@example.com"
}
```

কিন্তু file upload করার সময় সাধারণত ব্যবহার হয়:

```http
Content-Type: multipart/form-data
```

কারণ file binary data হিসেবে পাঠানো হয়।

---

## NestJS-এ Multer

NestJS `FileInterceptor()` ব্যবহার করে Multer-এর functionality সহজভাবে ব্যবহার করা যায়।

ধরুন:

```http
POST /users/profile-image
```

Controller:

```ts
import {
  Controller,
  Post,
  UploadedFile,
  UseInterceptors,
} from '@nestjs/common';

import { FileInterceptor } from '@nestjs/platform-express';

@Controller('users')
export class UsersController {
  @Post('profile-image')
  @UseInterceptors(FileInterceptor('image'))
  uploadProfileImage(
    @UploadedFile() file: Express.Multer.File,
  ) {
    console.log(file);

    return {
      message: 'File uploaded successfully',
      filename: file.originalname,
    };
  }
}
```

এখানে সবচেয়ে গুরুত্বপূর্ণ:

```ts
FileInterceptor('image')
```

`image` হচ্ছে frontend থেকে পাঠানো field name।

Frontend:

```html
<input type="file" name="image" />
```

অর্থাৎ:

```text
Frontend                  Backend

name="image"   ────────>  FileInterceptor("image")
                                  │
                                  ▼
                           @UploadedFile()
```

---

## `file` এর মধ্যে কী থাকে?

ধরুন `profile.jpg` upload করলেন।

```ts
console.log(file);
```

তাহলে প্রায় এরকম information পাবেন:

```ts
{
  fieldname: 'image',
  originalname: 'profile.jpg',
  encoding: '7bit',
  mimetype: 'image/jpeg',
  size: 245678,
  buffer: <Buffer ...>
}
```

সবচেয়ে গুরুত্বপূর্ণ properties:

```ts
file.originalname
```

Original filename:

```text
profile.jpg
```

---

```ts
file.mimetype
```

File-এর MIME type:

```text
image/jpeg
```

---

```ts
file.size
```

File size bytes-এ:

```text
245678
```

---

```ts
file.buffer
```

File-এর actual binary data।

Cloudinary/R2-এর মতো storage-এ upload করার সময় এটা কাজে লাগে।

---

# 2. File Validation

User যেকোনো file upload করতে পারে।

যেমন আপনার profile image-এর জন্য user পাঠিয়ে দিল:

```text
virus.exe
```

অথবা:

```text
movie.mp4
```

অথবা বিশাল:

```text
500 MB image
```

তাই server-এ file validate করতে হবে।

মূলত আমরা check করি:

```text
File
 │
 ├── File আছে?
 │
 ├── MIME type ঠিক?
 │
 ├── Size ঠিক?
 │
 └── সব ঠিক হলে
          ↓
       Upload
```

---

## `ParseFilePipe`

NestJS-এ file validation করার সুন্দর উপায়:

```ts
@UploadedFile(
  new ParseFilePipe({
    validators: [
      new MaxFileSizeValidator({
        maxSize: 5 * 1024 * 1024,
      }),
    ],
  }),
)
file: Express.Multer.File
```

এখানে:

```ts
5 * 1024 * 1024
```

মানে:

```text
5 MB
```

---

## Complete example

```ts
import {
  Controller,
  Post,
  UploadedFile,
  UseInterceptors,
  ParseFilePipe,
  MaxFileSizeValidator,
} from '@nestjs/common';

import { FileInterceptor } from '@nestjs/platform-express';

@Controller('users')
export class UsersController {
  @Post('profile-image')
  @UseInterceptors(FileInterceptor('image'))
  uploadProfileImage(
    @UploadedFile(
      new ParseFilePipe({
        validators: [
          new MaxFileSizeValidator({
            maxSize: 5 * 1024 * 1024,
          }),
        ],
      }),
    )
    file: Express.Multer.File,
  ) {
    return {
      message: 'Image uploaded',
      filename: file.originalname,
      size: file.size,
    };
  }
}
```

এখন 5 MB-এর বেশি file দিলে request reject হবে।

---

# 3. MIME Type

এটা খুব গুরুত্বপূর্ণ।

## MIME type কী?

MIME type server-কে বলে file-এর **content type কী ধরনের**।

উদাহরণ:

```text
.jpg
→ image/jpeg

.png
→ image/png

.webp
→ image/webp

.pdf
→ application/pdf

.mp4
→ video/mp4

.json
→ application/json
```

তাই যদি আপনি শুধু image allow করতে চান:

```text
image/jpeg
image/png
image/webp
```

allow করতে পারেন।

---

## MIME validation

NestJS:

```ts
import { FileTypeValidator } from '@nestjs/common';
```

তারপর:

```ts
new FileTypeValidator({
  fileType: /^image\/(jpeg|png|webp)$/,
})
```

Complete:

```ts
@UploadedFile(
  new ParseFilePipe({
    validators: [
      new MaxFileSizeValidator({
        maxSize: 5 * 1024 * 1024,
      }),

      new FileTypeValidator({
        fileType: /^image\/(jpeg|png|webp)$/,
      }),
    ],
  }),
)
file: Express.Multer.File
```

এখন:

```text
JPEG  ✅
PNG   ✅
WEBP  ✅

PDF   ❌
MP4   ❌
EXE   ❌
```

---

## একটা গুরুত্বপূর্ণ ব্যাপার

শুধু filename দেখে বিশ্বাস করা উচিত না:

```text
virus.exe → rename → photo.jpg
```

Filename `.jpg` হলেই সেটা সত্যিকারের image—এটা ধরে নেওয়া নিরাপদ নয়।

তাই production application-এ file type validation carefully করতে হয়।

---

# 4. File Size Limits

ধরুন আপনার application-এ profile image maximum:

```text
5 MB
```

তাহলে server-এ limit দিতে হবে।

```ts
new MaxFileSizeValidator({
  maxSize: 5 * 1024 * 1024,
})
```

---

## কেন size limit দরকার?

ধরুন attacker বারবার:

```text
500 MB
500 MB
500 MB
500 MB
500 MB
```

upload করছে।

Server-এর:

```text
RAM
CPU
Network
Storage
```

সবকিছুর উপর চাপ পড়বে।

তাই upload system-এ সাধারণত:

```text
Maximum file size
Maximum number of files
Allowed file types
```

define করা হয়।

---

# 5. Multiple File Upload

একটা file:

```ts
FileInterceptor()
```

একাধিক file:

```ts
FilesInterceptor()
```

Example:

```ts
@Post('product-images')
@UseInterceptors(
  FilesInterceptor('images', 5),
)
uploadProductImages(
  @UploadedFiles()
  files: Express.Multer.File[],
) {
  return {
    count: files.length,
  };
}
```

এখানে:

```ts
FilesInterceptor('images', 5)
```

মানে maximum:

```text
5 files
```

Frontend:

```text
images[]:
    image1.jpg
    image2.jpg
    image3.jpg
```

---

# 6. Cloudinary

এখন আসি actual production storage-এর দিকে।

Local server-এ file রাখা technically সম্ভব:

```text
NestJS
  │
  ▼
uploads/
   ├── image1.jpg
   ├── image2.jpg
   └── image3.jpg
```

কিন্তু production application-এ এটা সাধারণত ভালো architecture না।

কারণ server replace/redeploy হলে local files হারানোর সমস্যা হতে পারে এবং application server-এর disk storage-এর সাথে user uploads মিশে যায়।

তাই external object/media storage ব্যবহার করা হয়।

Cloudinary বিশেষভাবে image/video management-এর জন্য জনপ্রিয়।

---

## Cloudinary কী?

সহজ ভাষায়:

> **Cloudinary হলো cloud-based media storage + image/video processing service।**

আপনি image upload করবেন:

```text
NestJS
   │
   │ image
   ▼
Cloudinary
   │
   ├── Store
   ├── Transform
   ├── Resize
   ├── Compress
   └── CDN delivery
```

তারপর Cloudinary আপনাকে URL দিতে পারে:

```text
https://.../profile-image.webp
```

Database-এ সাধারণত image-এর binary data রাখবেন না।

বরং রাখবেন:

```ts
{
  avatarUrl: "https://...",
}
```

---

## NestJS + Cloudinary ধারণা

Flow:

```text
Client
   │
   │ multipart/form-data
   ▼
NestJS
   │
   ├── MIME validation
   ├── Size validation
   │
   ▼
Cloudinary
   │
   ▼
URL
   │
   ▼
MongoDB
```

ধরুন user profile:

```ts
{
  name: "Aziz",
  email: "aziz@example.com",
  avatarUrl: "https://cloudinary.com/..."
}
```

Database-এ image binary রাখার দরকার নেই।

---

# 7. Cloudflare R2

Cloudinary আর Cloudflare R2 একই জিনিস না।

Cloudflare R2 হলো **object storage**।

এটা অনেকটা:

```text
Amazon S3
        ↓
Object Storage
```

এর মতো।

R2-তে আপনি রাখতে পারেন:

```text
profile.jpg
product-1.jpg
product-2.webp
invoice.pdf
course.pdf
```

Structure:

```text
R2 Bucket
│
├── users/
│   ├── user-1/avatar.webp
│   └── user-2/avatar.webp
│
├── products/
│   ├── product-1/image-1.webp
│   └── product-1/image-2.webp
│
└── documents/
    └── invoice.pdf
```

---

# Cloudinary বনাম R2

সহজভাবে:

| বিষয়                  | Cloudinary       | Cloudflare R2               |
| --------------------- | ---------------- | --------------------------- |
| মূল কাজ               | Media management | Object storage              |
| Image transformation  | ✅                | নিজে করতে হবে               |
| Image resize          | ✅                | নিজে করতে হবে               |
| Image optimization    | ✅                | আলাদা processing লাগতে পারে |
| PDF storage           | ✅                | ✅                           |
| সাধারণ file storage   | ভালো             | খুব ভালো                    |
| S3-compatible         | মূলত না          | ✅                           |
| CDN-oriented delivery | ✅                | Cloudflare ecosystem        |
| Raw object storage    | কম focus         | মূল focus                   |

যেমন আপনার application যদি হয়:

```text
Product images
Profile pictures
Image resizing
Image optimization
```

তাহলে Cloudinary-এর functionality বেশ useful।

আর যদি হয়:

```text
PDF
ZIP
Documents
Videos
Large files
General object storage
```

তাহলে R2-এর object-storage model খুব useful।

---

# 8. Presigned URLs

এটা production file upload-এর সবচেয়ে গুরুত্বপূর্ণ conceptগুলোর একটি।

ধরুন 20 MB-এর একটা image upload করতে হবে।

একটা approach:

```text
Browser
   │
   │ 20 MB
   ▼
NestJS
   │
   │ 20 MB
   ▼
R2
```

এখানে NestJS server শুধু file receive করে আবার storage-এ পাঠাচ্ছে।

মানে:

```text
Browser
   ↓
NestJS
   ↓
R2
```

বড় file হলে application server-এর উপর unnecessary load পড়ে।

---

## Presigned URL দিয়ে কী হয়?

এবার architecture:

```text
Browser
   │
   │ "I want to upload a file"
   ▼
NestJS
   │
   │ generate signed URL
   ▼
Browser
   │
   │ directly upload
   ▼
R2
```

অর্থাৎ file NestJS server-এর মধ্য দিয়ে যাচ্ছে না।

---

## Step-by-step

### Step 1

Frontend বলল:

```http
POST /uploads/presigned-url
```

সাথে:

```json
{
  "filename": "profile.jpg",
  "contentType": "image/jpeg"
}
```

---

### Step 2

NestJS R2-এর জন্য একটা temporary signed URL তৈরি করবে।

Response:

```json
{
  "uploadUrl": "https://....",
  "key": "users/123/profile.jpg"
}
```

---

### Step 3

Browser সরাসরি R2-তে upload করবে:

```text
Browser
   │
   │ PUT uploadUrl
   ▼
Cloudflare R2
```

---

### Step 4

Upload complete।

তারপর database-এ key/reference রাখা যায়:

```ts
{
  avatarKey: "users/123/profile.jpg"
}
```

---

## Presigned URL কেন "signed"?

কারণ URL-টা সাধারণ public upload URL না।

এটা সাধারণত:

```text
temporary
+
authorized
+
limited
```

হয়।

যেমন:

```text
Valid for 10 minutes
```

অথবা:

```text
Valid for specific object
```

তাই user আপনার R2 credentials পাবে না।

---

# 9. Presigned URL Flow — Production Architecture

ধরুন product image upload:

```text
                    ┌──────────────┐
                    │    Client    │
                    └──────┬───────┘
                           │
                    1. Request URL
                           │
                           ▼
                    ┌──────────────┐
                    │    NestJS    │
                    └──────┬───────┘
                           │
                    2. Generate URL
                           │
                           ▼
                    ┌──────────────┐
                    │ Cloudflare R2│
                    └──────────────┘
                           ▲
                           │
                    3. Direct Upload
                           │
                    ┌──────┴───────┐
                    │    Client    │
                    └──────────────┘
```

এখানে NestJS-এর কাজ:

```text
Authentication
Authorization
Validation
Generate signed URL
Save metadata
```

আর R2-এর কাজ:

```text
Store file
```

---

# 10. Image Optimization

ধরুন user upload করল:

```text
original.jpg
4000 × 3000
8 MB
```

Website-এ দরকার:

```text
800 × 600
150 KB
```

যদি original image সরাসরি user-এর browser-এ পাঠান:

```text
8 MB
```

তাহলে unnecessary bandwidth ব্যবহার হবে।

Image optimization-এর উদ্দেশ্য:

```text
Large image
     ↓
Resize
     ↓
Compress
     ↓
Modern format
     ↓
Smaller image
```

---

## Example

Original:

```text
4000 × 3000
8 MB
JPEG
```

Optimized:

```text
1200 × 900
180 KB
WebP
```

Visual quality প্রায় একই রাখা যায়, কিন্তু file অনেক ছোট।

---

# 11. Resize

ধরুন product image:

```text
5000 × 4000
```

আপনার website-এ maximum দরকার:

```text
1200 × 1200
```

তাহলে resize করা যায়।

Conceptually:

```ts
resize(1200, 1200)
```

Image processing-এর জন্য Node ecosystem-এ `sharp` খুব commonly used।

Example:

```ts
import sharp from 'sharp';

const optimizedImage = await sharp(file.buffer)
  .resize(1200, 1200, {
    fit: 'inside',
    withoutEnlargement: true,
  })
  .webp({
    quality: 80,
  })
  .toBuffer();
```

এখানে:

```ts
.resize(1200, 1200)
```

image ছোট করছে।

```ts
.webp({
  quality: 80,
})
```

WebP format এবং quality 80 ব্যবহার করছে।

```ts
.toBuffer()
```

optimized image আবার buffer হিসেবে দিচ্ছে।

---

# 12. Image Optimization কেন গুরুত্বপূর্ণ?

ধরুন 1000 user আপনার website visit করল।

Original image:

```text
8 MB
```

আর optimized:

```text
200 KB
```

তাহলে প্রতিবার 8 MB পাঠানোর পরিবর্তে প্রায় 200 KB পাঠানো সম্ভব।

ফলে:

```text
Bandwidth ↓
Page load time ↓
Storage ↓
Mobile data usage ↓
User experience ↑
```

---

# 13. একটা Complete NestJS Upload Example

এখন সব basic concept একসাথে দেখি।

ধরুন:

```http
POST /users/avatar
```

শুধু:

```text
JPEG
PNG
WEBP
```

allow করব।

Maximum:

```text
5 MB
```

Code:

```ts
import {
  Controller,
  Post,
  UploadedFile,
  UseInterceptors,
  ParseFilePipe,
  MaxFileSizeValidator,
  FileTypeValidator,
} from '@nestjs/common';

import { FileInterceptor } from '@nestjs/platform-express';

@Controller('users')
export class UsersController {
  @Post('avatar')
  @UseInterceptors(
    FileInterceptor('image'),
  )
  uploadAvatar(
    @UploadedFile(
      new ParseFilePipe({
        validators: [
          new MaxFileSizeValidator({
            maxSize: 5 * 1024 * 1024,
          }),

          new FileTypeValidator({
            fileType: /^image\/(jpeg|png|webp)$/,
          }),
        ],
      }),
    )
    file: Express.Multer.File,
  ) {
    return {
      message: 'Avatar uploaded successfully',

      file: {
        originalName: file.originalname,
        mimeType: file.mimetype,
        size: file.size,
      },
    };
  }
}
```

এখানে পুরো flow:

```text
             Client
                │
                │ multipart/form-data
                ▼
        FileInterceptor
                │
                ▼
             Multer
                │
                ▼
        ┌───────────────┐
        │ File Validate │
        └───────┬───────┘
                │
        ┌───────┴────────┐
        │                │
     MIME check       Size check
        │                │
        └───────┬────────┘
                │
                ▼
          Valid File
                │
                ▼
      Image Optimization
                │
                ▼
       Cloudinary / R2
                │
                ▼
          File URL/Key
                │
                ▼
            Database
```

---

# 14. Production File Upload-এর Mental Model

এই পুরো chapter-টা একটা formula হিসেবে মনে রাখুন:

```text
                 FILE UPLOAD
                      │
       ┌──────────────┼──────────────┐
       │              │              │
     Receive       Validate        Store
       │              │              │
    Multer         MIME Type     Cloudinary
                   File Size        R2
                      │
                      ▼
                Optimization
                      │
                    Sharp
                      │
                      ▼
              Database stores
                URL / Key
```

আর বড় file-এর ক্ষেত্রে:

```text
Normal:

Client
  ↓
NestJS
  ↓
Storage


Presigned:

Client
  ↓
NestJS → Signed URL
  ↓
Client
  ↓
Storage
```

**সবচেয়ে গুরুত্বপূর্ণ distinction:**

* **Multer** → file receive করার জন্য
* **File validation** → file গ্রহণযোগ্য কিনা যাচাই করার জন্য
* **MIME type** → file কী ধরনের তা যাচাই করার জন্য
* **File size limit** → file কত বড় হতে পারবে তা control করার জন্য
* **Cloudinary** → image/video storage + transformation-এর জন্য
* **Cloudflare R2** → general-purpose object storage-এর জন্য
* **Presigned URL** → client-কে সরাসরি storage-এ upload করার সুযোগ দেওয়ার জন্য
* **Image optimization** → image resize/compress/modern format করার জন্য
