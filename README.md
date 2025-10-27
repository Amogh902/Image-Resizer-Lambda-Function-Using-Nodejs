# Serverless Image Resizer Project

A **serverless image resizing system** built on **AWS S3**, **AWS Lambda**, and **Node.js (Sharp)**.
When an image is uploaded to the S3 bucket, it automatically triggers a Lambda function that resizes the image and saves it back to S3.

---

## Architecture Overview

```
S3 Bucket (originals/)  →  EventBridge/S3 Trigger  →  Lambda (Node.js + Sharp)  →  S3 (resized/<size>/)
```

### Components

* **Amazon S3**: Stores both the original and resized images.
* **AWS Lambda**: Executes resizing logic using Sharp.
* **IAM Role**: Provides Lambda permission to access S3 and CloudWatch.
* **CloudWatch Logs**: Monitors Lambda execution.

![](/Image-resizer-img/Architecture-img.png)

---

## Step 1 — Create S3 Bucket

1. Go to **AWS Management Console → S3 → Create bucket**
2. Bucket name: `serverless-image-resizer-project-bucket`
3. Create two folders inside it:

   * `originals/` → Upload original images here.
   * `resized/` → Lambda will save processed images here.

![](/Image-resizer-img/Bucket-img.png)

---

## Step 2 — Create IAM Role

1. Go to **IAM → Roles → Create Role**
2. Select **AWS Service → Lambda**
3. Attach the following **AWS Managed Policy**:

   * **`CloudWatchLogsFullAccess`** — allows Lambda to write logs to CloudWatch.
4. Attach the following **custom inline policy** to grant S3 permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::serverless-image-resizer-project-bucket",
        "arn:aws:s3:::serverless-image-resizer-project-bucket/originals/*",
        "arn:aws:s3:::serverless-image-resizer-project-bucket/resized/*"
      ]
    }
  ]
}
```

![](/Image-resizer-img/IAM-role-img.png)

---

## Step 3 — Create Lambda Function

1. Go to **Lambda → Create function**
2. Runtime: **Node.js 18.x**
3. Name: `image-resizer-function`
4. Assign role: `IAM-image-resizer-Role`
5. Set the following **Environment Variables:**

| Key           | Value                                   |
| ------------- | --------------------------------------- |
| BUCKET_NAME   | serverless-image-resizer-project-bucket |
| OUTPUT_PREFIX | resized                                 |
| SIZES         | 800x600                                 |

![](/Image-resizer-img/Lambda-fuction-img1.png)

---

## Step 4 — Package Dependencies (Sharp)

On a **Linux/EC2** environment (important for compatibility):

```bash
mkdir image-resizer-node
cd image-resizer-node
npm init -y
npm install sharp aws-sdk
```

Add your code in `index.mjs`:

```javascript
import AWS from "aws-sdk";
import Sharp from "sharp";

const s3 = new AWS.S3();

export const handler = async (event) => {
  const record = event.Records[0];
  const bucket = process.env.BUCKET_NAME;
  const key = decodeURIComponent(record.s3.object.key.replace(/\+/g, " "));

  if (key.startsWith("resized/")) return { status: "skipped" };

  const outputPrefix = process.env.OUTPUT_PREFIX || "resized";
  const sizes = (process.env.SIZES || "800x600").split(",");

  try {
    const originalImage = await s3.getObject({ Bucket: bucket, Key: key }).promise();

    for (const size of sizes) {
      const [width, height] = size.toLowerCase().split("x").map(Number);

      const resizedImage = await Sharp(originalImage.Body)
        .resize(width, height, { fit: "inside" })
        .toBuffer();

      const fileName = key.split("/").pop();
      const newKey = `${outputPrefix}/${size}/${fileName}`;

      await s3.putObject({
        Bucket: bucket,
        Key: newKey,
        Body: resizedImage,
        ContentType: "image/jpeg",
      }).promise();

      console.log(`✅ Uploaded resized image to ${newKey}`);
    }

    return { status: "success" };
  } catch (err) {
    console.error("❌ Error processing image:", err);
    throw err;
  }
};
```

Then zip it:

```bash
zip -r ../lambda_function.zip .
```

Upload `lambda_function.zip` to Lambda under **Code → Upload from .zip file**.

![](/Image-resizer-img/Lambda-fuction-img2.png)

---

## Step 5 — Add S3 Trigger

1. Go to **Lambda → Configuration → Triggers → Add trigger**
2. Choose **S3**
3. Event type: `All object create events`
4. Prefix: `originals/`

![](/Image-resizer-img/Trigger-img.png)

---

## Step 6 — Test the Setup

1. Upload any image (e.g., `photo.jpg`) to:

   ```
   s3://serverless-image-resizer-project-bucket/originals/
   ```
2. Wait a few seconds.
3. Check for the resized image in:

   ```
   s3://serverless-image-resizer-project-bucket/resized/800x600/photo.jpg
   ```

![](/Image-resizer-img/Img-uploaded-to-originals-img.png)

![](/Image-resizer-img/resized-images-img.png)

---

## Step 7 — Monitor with CloudWatch

1. Go to **CloudWatch → Logs → Log groups → /aws/lambda/image-resizer-function**
2. Open the latest log stream.
3. You should see:

   ```
   Uploaded resized image to resized/800x600/photo.jpg
   ```

![](/Image-resizer-img/Cloud-watch-log-img.png)

---

## Cost Considerations

| Service    | Cost Type | Notes                          |
| ---------- | --------- | ------------------------------ |
| S3         | Storage   | Only pay for images stored     |
| Lambda     | Compute   | Charged per request + duration |
| CloudWatch | Logs      | Minimal unless logs are large  |

No charges occur if no images are uploaded or Lambda isn’t invoked.

---

## Key Learnings

* AWS Lambda seamlessly handles image processing when triggered by S3 events.
* Sharp is a fast, reliable image processing library for Node.js.
* IAM permissions must explicitly include `s3:ListBucket`, `s3:GetObject`, and `s3:PutObject`.
* CloudWatchLogsFullAccess ensures proper Lambda log streaming.
* Serverless = scalable, cost-effective, and maintenance-free.

---

## Project Summary

| Component       | Status       |
| --------------- | ------------ |
| S3 Bucket       | ✅ Created    |
| IAM Role        | ✅ Configured |
| Lambda          | ✅ Working    |
| Trigger         | ✅ Active     |
| CloudWatch Logs | ✅ Verified   |

**Congratulations!** You’ve successfully built a serverless image resizer with AWS Lambda + Node.js.
