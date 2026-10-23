# multer_multer_storage_cloudinary

```jsx
/* eslint-disable @typescript-eslint/no-explicit-any */
import { v2 as cloudinary } from "cloudinary";
import { env } from "./env";
import { CloudinaryStorage } from "multer-storage-cloudinary";
import multer from "multer";

cloudinary.config({
  cloud_name: env.CLOUDINARY_CLOUD_NAME,
  api_key: env.CLOUDINARY_API_KEY,
  api_secret: env.CLOUDINARY_API_SECRET,
});

export const cloudinaryUploader = cloudinary;

const storage = new CloudinaryStorage({
  cloudinary: cloudinaryUploader as any,

  params: async (req, file) => {

    console.log("File received for upload:", file);
    const extension = file.originalname
      .split(".")
      .at(-1)
      ?.toLowerCase();

    const originalname = file.originalname
      .split(".")
      .slice(0, -1)
      .join(".");

    const standardizedFileName = originalname
      .replace(/\s+/g, "_")
      .replace(/[^a-zA-Z0-9_-]/g, "")
      .toLowerCase();

    const public_id = `${standardizedFileName}_${Math.random()
      .toString(36)
      .slice(2, 5)}_${Date.now()}`;

    // Root folder = project name
    const projectName = "ChakriOva";

    const fileTypeFolders: Record<string, string> = {
      pdf: "pdfs",
      doc: "docs",
      docx: "docs",
      txt: "documents",
      csv: "documents",
      xls: "spreadsheets",
      xlsx: "spreadsheets",
      ppt: "presentations",
      pptx: "presentations",
      jpg: "images",
      jpeg: "images",
      png: "images",
      webp: "images",
      gif: "images",
      svg: "images",
      mp3: "audios",
      wav: "audios",
      mp4: "videos",
      webm: "videos",
      zip: "archives",
      rar: "archives",
      "7z": "archives",
    };
    const fileTypeFolder =
      fileTypeFolders[extension?.toLowerCase() ?? ""] ?? "others";


    // project-name/file-type
    const folderName = `${projectName}/${fileTypeFolder}`;

    console.log("body:", req.body);

    if (req.body.data) {
      req.body = JSON.parse(req.body.data);
    }

    console.log("body:", req.body);

    console.log("Upload Details:", {
      file,
      extension,
      originalname,
      standardizedFileName,
      public_id,
      folderName,
    });

    return {
      folder: folderName,
      public_id,
      resource_type: "auto",
    };
  },
});

export const multerUploader = multer({
  storage,
});


export const cloudinaryDelete = async (url: string) => {
  const publicId = url.split("/").slice(-3).join("/").split(".")[0];

  console.log("Deleting file from Cloudinary with publicId:", url, publicId);

  const result = await cloudinaryUploader.uploader.destroy(
    publicId,
    // { resource_type:  }
  );
  console.log("Cloudinary delete result:", result);
}
```
