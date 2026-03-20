# Image Uploads on Hive

## Primary Service: images.hive.blog

The official Hive image hosting service. Free, permanent hosting tied to your Hive account.

### How It Works

1. **Create signing challenge** — concatenate `ImageSigningChallenge` prefix + raw file bytes
2. **Sign with posting key** — via Hive Keychain or direct key signing
3. **POST to** `https://images.hive.blog/{username}/{signature}` with file as multipart form data
4. **Response** — JSON with `{ url: "https://images.hive.blog/DQm.../filename.ext" }`

### Browser Implementation (Hive Keychain)

```javascript
async function uploadToHiveBlog(file, username) {
  if (!window.hive_keychain) throw new Error('Hive Keychain required');

  // 1. Read file into bytes
  const fileBytes = await new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => resolve(new Uint8Array(reader.result));
    reader.onerror = () => reject(reader.error);
    reader.readAsArrayBuffer(file);
  });

  // 2. Build challenge: "ImageSigningChallenge" + file content
  const prefix = new TextEncoder().encode('ImageSigningChallenge');
  const challenge = new Uint8Array(prefix.length + fileBytes.length);
  challenge.set(prefix, 0);
  challenge.set(fileBytes, prefix.length);

  // 3. Keychain expects the challenge as JSON stringified array
  const challengeString = JSON.stringify(Array.from(challenge));

  // 4. Sign with posting key via Keychain
  const signature = await new Promise((resolve, reject) => {
    window.hive_keychain.requestSignBuffer(
      username,
      challengeString,
      'Posting',
      (response) => {
        if (response.success && response.result) resolve(response.result);
        else reject(new Error(response.message || 'Signing cancelled'));
      }
    );
  });

  // 5. Upload
  const formData = new FormData();
  formData.append('file', file);
  const resp = await fetch(`https://images.hive.blog/${username}/${signature}`, {
    method: 'POST',
    body: formData
  });
  if (!resp.ok) throw new Error(`Upload failed: ${resp.status}`);
  const data = await resp.json();
  if (!data.url) throw new Error('No URL in response');
  return data.url;
}
```

### Node.js / Server-Side Implementation (Direct Key Signing)

When the server holds the posting key (e.g. for automated uploads):

```javascript
const { PrivateKey, cryptoUtils } = require('@hiveio/dhive');
const fs = require('fs');

async function signAndUpload(filePath, username, postingKeyWIF) {
  const fileContent = fs.readFileSync(filePath);
  const prefix = Buffer.from('ImageSigningChallenge');
  const challenge = Buffer.concat([prefix, fileContent]);

  // Hash and sign
  const hash = cryptoUtils.sha256(challenge);
  const privateKey = PrivateKey.fromString(postingKeyWIF);
  const signature = privateKey.sign(hash).toString();

  const formData = new FormData();
  formData.append('file', new Blob([fileContent]), path.basename(filePath));

  const resp = await fetch(`https://images.hive.blog/${username}/${signature}`, {
    method: 'POST',
    body: formData
  });
  const data = await resp.json();
  return data.url;
}
```

### Server-Side Proxy (Recommended for Web Apps)

To avoid CORS issues and keep API keys secure, proxy uploads through your backend:

```javascript
// Client sends: { file, username, signature } to your API
// Server tries images.hive.blog first, falls back to 3speak

async function handleUpload(file, username, signature) {
  // Try primary: images.hive.blog
  if (username && signature) {
    try {
      const formData = new FormData();
      formData.append('file', file);
      const resp = await fetch(`https://images.hive.blog/${username}/${signature}`, {
        method: 'POST', body: formData
      });
      if (resp.ok) {
        const data = await resp.json();
        return { success: true, url: data.url };
      }
    } catch (e) {
      console.warn('Hive.blog upload failed, trying fallback:', e.message);
    }
  }

  // Fallback: images.3speak.tv
  const formData = new FormData();
  formData.append('image', file);
  const resp = await fetch('https://images.3speak.tv/upload', {
    method: 'POST',
    headers: { 'Authorization': `Bearer ${IMAGE_SERVER_API_KEY}` },
    body: formData
  });
  const data = await resp.json();
  if (!data.success || !data.url) throw new Error('3Speak upload failed');
  return { success: true, url: data.url, source: '3speak' };
}
```

## Fallback Service: images.3speak.tv

When images.hive.blog is down or returns errors.

- **Endpoint:** `POST https://images.3speak.tv/upload`
- **Auth:** `Authorization: Bearer {API_KEY}` header
- **Field name:** `image` (not `file`)
- **Response:** `{ success: boolean, url: string }`
- **API key:** Contact 3Speak team or use `NEXT_PUBLIC_IMAGE_SERVER_API_KEY` env var

## HEIC/HEIF Conversion (iOS Devices)

**Problem:** iOS cameras produce HEIC images by default. `images.hive.blog` accepts them, but many browsers and Hive frontends **cannot display HEIC images**. The upload succeeds but the image appears broken everywhere.

**Solution:** Convert HEIC to JPEG client-side before uploading.

### Browser (Canvas method)

```javascript
async function convertHeicToJpeg(file) {
  // Only convert HEIC/HEIF
  if (!/^image\/(heic|heif)$/i.test(file.type) && !/\.heic$/i.test(file.name)) {
    return file;
  }

  return new Promise((resolve, reject) => {
    const img = new Image();
    const url = URL.createObjectURL(file);
    img.onload = () => {
      const canvas = document.createElement('canvas');
      canvas.width = img.naturalWidth;
      canvas.height = img.naturalHeight;
      canvas.getContext('2d').drawImage(img, 0, 0);
      URL.revokeObjectURL(url);
      canvas.toBlob(
        (blob) => {
          if (!blob) return reject(new Error('Canvas conversion failed'));
          resolve(new File([blob], file.name.replace(/\.heic$/i, '.jpg'), { type: 'image/jpeg' }));
        },
        'image/jpeg',
        0.9
      );
    };
    img.onerror = () => { URL.revokeObjectURL(url); reject(new Error('Failed to decode image')); };
    img.src = url;
  });
}
```

**Note:** The Canvas method only works if the browser can decode HEIC. Safari can; Chrome/Firefox cannot natively. For cross-browser HEIC support, use `heic2any` library:

```javascript
import heic2any from 'heic2any';

async function convertHeicToJpeg(file) {
  if (!/^image\/(heic|heif)$/i.test(file.type) && !/\.heic$/i.test(file.name)) {
    return file;
  }
  const blob = await heic2any({ blob: file, toType: 'image/jpeg', quality: 0.9 });
  return new File([blob], file.name.replace(/\.heic$/i, '.jpg'), { type: 'image/jpeg' });
}
```

### React Native (Expo)

```javascript
import * as ImageManipulator from 'expo-image-manipulator';

async function normalizeImage(uri, mimeType) {
  if (mimeType === 'image/heic' || mimeType === 'image/heif' || uri.startsWith('ph://')) {
    const result = await ImageManipulator.manipulateAsync(
      uri,
      [], // no transforms
      { compress: 0.8, format: ImageManipulator.SaveFormat.JPEG }
    );
    return result.uri; // file:// URI, JPEG format
  }
  return uri;
}
```

## Gotchas & Battle-Tested Lessons

### 1. Challenge format is NOT just the username
The signing challenge is `'ImageSigningChallenge' + fileContent` (raw bytes), NOT the username or a simple string. Getting this wrong produces a valid signature that the server rejects.

### 2. Keychain expects JSON stringified buffer
When using `requestSignBuffer`, pass `JSON.stringify(Array.from(challengeBuffer))`. Keychain hashes this internally before signing.

### 3. Field name differs between services
- **images.hive.blog:** field name is `file`
- **images.3speak.tv:** field name is `image`
Mix these up and you get a silent failure or 400 error.

### 4. CORS can block direct browser uploads
`images.hive.blog` may reject cross-origin requests from some domains. If you hit CORS errors, proxy through your backend.

### 5. images.hive.blog occasionally goes down
Always implement a fallback. The 3Speak image server is the most common alternative. Local storage is also viable as a last resort.

### 6. Uploaded images are permanent
There is no delete API. Once uploaded, images stay on the CDN indefinitely. This is a feature for blockchain content but means you should validate/compress before uploading.

### 7. Large files may timeout
For images over 5MB, consider client-side compression before upload. The `canvas.toBlob` quality parameter (0.7-0.85) usually produces good results at much smaller file sizes.

### 8. Video thumbnails
For video posts, extract a frame client-side and upload as the thumbnail:

```javascript
function extractVideoThumbnail(videoUrl) {
  return new Promise((resolve) => {
    const video = document.createElement('video');
    video.crossOrigin = 'anonymous';
    video.src = videoUrl;
    video.currentTime = 0.5; // grab frame at 0.5s
    video.oncanplay = () => {
      const canvas = document.createElement('canvas');
      canvas.width = video.videoWidth;
      canvas.height = video.videoHeight;
      canvas.getContext('2d').drawImage(video, 0, 0);
      canvas.toBlob((blob) => resolve(blob), 'image/jpeg', 0.9);
    };
  });
}
```
