# multer_multer_storage_cloudinary

```jsx
/* eslint-disable @typescript-eslint/no-explicit-any */

import { v2 as cloudinary } from "cloudinary";
import { env } from "./env";
import { CloudinaryStorage } from "multer-storage-cloudinary";
import multer from "multer";
import path from "node:path";
import crypto from "node:crypto";

cloudinary.config({
  cloud_name: env.CLOUDINARY_CLOUD_NAME,
  api_key: env.CLOUDINARY_API_KEY,
  api_secret: env.CLOUDINARY_API_SECRET,
});

export const cloudinaryUploader = cloudinary;

// -------------------------------------
// File type configuration
// -------------------------------------

type CloudinaryResourceType = "image" | "video" | "raw";

const fileTypeConfig: Record<
  string,
  {
    folder: string;
    resourceType: CloudinaryResourceType;
  }
> = {
  // Images
  jpg: { folder: "images", resourceType: "image" },
  jpeg: { folder: "images", resourceType: "image" },
  png: { folder: "images", resourceType: "image" },
  webp: { folder: "images", resourceType: "image" },
  gif: { folder: "images", resourceType: "image" },
  svg: { folder: "images", resourceType: "image" },
  avif: { folder: "images", resourceType: "image" },
  ico: { folder: "images", resourceType: "image" },
  bmp: { folder: "images", resourceType: "image" },
  tif: { folder: "images", resourceType: "image" },
  tiff: { folder: "images", resourceType: "image" },

  // Documents
  pdf: { folder: "pdfs", resourceType: "raw" },
  doc: { folder: "docs", resourceType: "raw" },
  docx: { folder: "docs", resourceType: "raw" },
  txt: { folder: "documents", resourceType: "raw" },
  csv: { folder: "documents", resourceType: "raw" },
  json: { folder: "documents", resourceType: "raw" },
  xml: { folder: "documents", resourceType: "raw" },

  // Spreadsheets
  xls: { folder: "spreadsheets", resourceType: "raw" },
  xlsx: { folder: "spreadsheets", resourceType: "raw" },
  ods: { folder: "spreadsheets", resourceType: "raw" },

  // Presentations
  ppt: { folder: "presentations", resourceType: "raw" },
  pptx: { folder: "presentations", resourceType: "raw" },
  odp: { folder: "presentations", resourceType: "raw" },

  // Audio
  mp3: { folder: "audios", resourceType: "video" },
  wav: { folder: "audios", resourceType: "video" },
  ogg: { folder: "audios", resourceType: "video" },
  m4a: { folder: "audios", resourceType: "video" },
  aac: { folder: "audios", resourceType: "video" },
  flac: { folder: "audios", resourceType: "video" },

  // Videos
  mp4: { folder: "videos", resourceType: "video" },
  webm: { folder: "videos", resourceType: "video" },
  mov: { folder: "videos", resourceType: "video" },
  avi: { folder: "videos", resourceType: "video" },
  mkv: { folder: "videos", resourceType: "video" },
  m4v: { folder: "videos", resourceType: "video" },

  // Archives
  zip: { folder: "archives", resourceType: "raw" },
  rar: { folder: "archives", resourceType: "raw" },
  "7z": { folder: "archives", resourceType: "raw" },
  tar: { folder: "archives", resourceType: "raw" },
  gz: { folder: "archives", resourceType: "raw" },
};

// -------------------------------------
// Generate safe public ID
// -------------------------------------

const generatePublicId = (originalName: string): string => {
  const name = path.parse(originalName).name;

  const safeName =
    name
      .normalize("NFKD")
      .replace(/[\u0300-\u036f]/g, "")
      .replace(/[^a-zA-Z0-9_-]+/g, "_")
      .replace(/^_+|_+$/g, "")
      .toLowerCase() || "file";

  return `${safeName}_${crypto.randomUUID()}`;
};

// -------------------------------------
// Cloudinary storage
// -------------------------------------

const storage = new CloudinaryStorage({
  cloudinary: cloudinaryUploader as any,

  params: async (_req, file) => {
    const extension = path.extname(file.originalname)
      .slice(1)
      .toLowerCase();

    const config = fileTypeConfig[extension] ?? {
      folder: "others",
      resourceType: "raw" as const,
    };

    const projectName = "ChakriOva";
    const folder = `${projectName}/${config.folder}`;
    const publicId = generatePublicId(file.originalname);

    return {
      folder,
      public_id: publicId,
      resource_type: config.resourceType,
      // Cloudinary public ID excludes the extension for this configuration.
      // Raw resources may retain the original filename extension in delivery URLs.
    };
  },
});

// -------------------------------------
// Multer uploader
// -------------------------------------

export const multerUploader = multer({
  storage,
  limits: {
    fileSize: 10 * 1024 * 1024, // 10 MB
    files: 5,
  },
});

// -------------------------------------
// Delete by public ID
// -------------------------------------

export const cloudinaryDeleteByPublicId = async (
  publicId: string,
  resourceType: CloudinaryResourceType = "image",
) => {
  const result = await cloudinaryUploader.uploader.destroy(publicId, {
    resource_type: resourceType,
    invalidate: true,
  });

  if (result.result !== "ok" && result.result !== "not found") {
    throw new Error(`Cloudinary deletion failed: ${result.result}`);
  }

  return result;
};

// -------------------------------------
// Delete by Cloudinary URL
// Best-effort parser for standard delivery URLs
// -------------------------------------

export const cloudinaryDeleteByUrl = async (url: string) => {
  let parsedUrl: URL;

  try {
    parsedUrl = new URL(url);
  } catch {
    throw new Error("Invalid file URL");
  }

  if (
    parsedUrl.protocol !== "https:" ||
    !parsedUrl.hostname.endsWith("cloudinary.com")
  ) {
    throw new Error("Invalid Cloudinary URL");
  }

  const segments = parsedUrl.pathname.split("/").filter(Boolean);
  const uploadIndex = segments.indexOf("upload");

  if (uploadIndex < 1) {
    throw new Error("Could not parse Cloudinary upload URL");
  }

  const resourceType = segments[uploadIndex - 1];

  if (
    resourceType !== "image" &&
    resourceType !== "video" &&
    resourceType !== "raw"
  ) {
    throw new Error("Unsupported Cloudinary resource type");
  }

  const publicIdParts = segments.slice(uploadIndex + 1);

  // Remove a version segment, e.g. v1730000000
  if (/^v\d+$/.test(publicIdParts[0] ?? "")) {
    publicIdParts.shift();
  }

  if (publicIdParts.length === 0) {
    throw new Error("Could not find Cloudinary public ID");
  }

  // Image/video delivery URLs normally contain an extension.
  // Raw public IDs may include the original extension.
  if (resourceType !== "raw") {
    const lastIndex = publicIdParts.length - 1;

    publicIdParts[lastIndex] = publicIdParts[lastIndex].replace(
      /\.[^.]+$/,
      "",
    );
  }

  const publicId = publicIdParts.join("/");

  return cloudinaryDeleteByPublicId(
    publicId,
    resourceType as CloudinaryResourceType,
  );
};
```
