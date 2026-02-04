# Deploy to S3 as a static website

Everything (HTML, map, chart, data) is static, so you can host it from a single S3 bucket.

## This project

- **Bucket:** `co-avalanche-fatalities`
- **Region:** `us-west-2`
- **Site URL:** http://co-avalanche-fatalities.s3-website-us-west-2.amazonaws.com

---

## Update by hand (manual deploy)

Use this when you change `index.html` or the GeoJSON and want to push updates to the live site.

1. **Upload `index.html`** to the bucket root:
   ```bash
   aws s3 cp index.html s3://co-avalanche-fatalities/index.html \
     --content-type "text/html" --region us-west-2 --profile work
   ```

2. **If you updated the GeoJSON**, gzip it, upload to `data/`, and set metadata (see [GeoJSON .gz workflow](#geojson-gz-workflow) below). If you didn’t change the data, skip this.

3. **Check the site:**  
   http://co-avalanche-fatalities.s3-website-us-west-2.amazonaws.com

---

## GeoJSON .gz workflow

The app loads `data/co_fatal_web.geojson.gz`. The file should be gzipped and served with `Content-Encoding: gzip` so the browser decompresses it (or the app decompresses in the browser as a fallback).

1. **Gzip the GeoJSON** (from the project root):
   ```bash
   gzip -k -9 data/co_fatal_web.geojson
   ```
   This creates `data/co_fatal_web.geojson.gz` and keeps the original.

2. **Upload the .gz file** to S3:
   ```bash
   aws s3 cp data/co_fatal_web.geojson.gz s3://co-avalanche-fatalities/data/co_fatal_web.geojson.gz \
     --region us-west-2 --profile work
   ```

3. **Set metadata** so S3 sends `Content-Encoding: gzip` (browser decompresses automatically). The console often doesn’t show an Edit option for metadata; use the CLI to copy the object back with new metadata:
   ```bash
   aws s3 cp s3://co-avalanche-fatalities/data/co_fatal_web.geojson.gz \
     s3://co-avalanche-fatalities/data/co_fatal_web.geojson.gz \
     --metadata-directive REPLACE \
     --content-type "application/geo+json" \
     --content-encoding gzip \
     --region us-west-2 --profile work
   ```

4. The app requests `data/co_fatal_web.geojson.gz`. If S3 doesn’t send the gzip header, the HTML includes a fallback that decompresses in the browser.

---

## First-time setup (AWS Console)

Use this when creating the bucket and enabling the site for the first time.

1. **Create bucket:** S3 → Create bucket → name `co-avalanche-fatalities`, region `us-west-2` → Create.

2. **Upload files** (keep the same paths):
   - Upload `index.html` to the bucket **root** (no folder).
   - Create a folder `data` and upload `co_fatal_web.geojson.gz` into it. Then set `Content-Encoding: gzip` via CLI (see [GeoJSON .gz workflow](#geojson-gz-workflow)); the console often doesn’t allow editing metadata.

3. **Turn on website hosting:**  
   Bucket → **Properties** → **Static website hosting** → Edit → Enable → Index document: `index.html` → Save.

4. **Make the bucket publicly readable:**  
   **Permissions** → **Block public access** → Edit → uncheck **Block all public access** → Save.  
   **Bucket policy** → Edit → paste:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "PublicReadGetObject",
         "Effect": "Allow",
         "Principal": "*",
         "Action": "s3:GetObject",
         "Resource": "arn:aws:s3:::co-avalanche-fatalities/*"
       }
     ]
   }
   ```
   Save.

5. **Open the website URL** under **Properties** → **Static website hosting**.

---

## HTTPS (optional)

The S3 website endpoint is **HTTP only**. For **HTTPS** and custom domains:

- Create a **CloudFront** distribution with the S3 website endpoint as origin.
- Use **Origin Access Control (OAC)** and turn off direct public access on the bucket; CloudFront will serve the files.
- Attach your domain and an ACM certificate to the distribution.

If you want, we can add a small CloudFront + OAC setup (e.g. Terraform or console steps) next.
