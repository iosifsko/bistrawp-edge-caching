# BistraWP: Full HTML Edge Caching with CloudFront + Lambda@Edge

This guide documents the completed setup for serving fully cached HTML pages for WordPress on the BistraWP platform using AWS CloudFront, Apache, and Lambda@Edge.

---

## ✅ Architecture Summary

- **EC2**: Ubuntu with Apache, WordPress installed
- **RDS**: MySQL database for WP
- **S3**: Media storage bucket
- **CloudFront**: CDN for static assets and full-page HTML
- **Apache**: Sends Cache-Control headers
- **Lambda@Edge**: Bypasses cache for logged-in users/admins
- **Certbot/Let’s Encrypt**: HTTPS enabled for `logs.bistrawp.io`

---

## ✅ Step-by-Step Configuration

### 🔹 1. Apache Cache-Control Headers

Edit Apache config:

```bash
sudo nano /etc/apache2/apache2.conf
```

Add:
```apache
<FilesMatch "\.(html|htm)$">
  Header set Cache-Control "public, max-age=86400"
</FilesMatch>
```

Then restart:
```bash
sudo systemctl restart apache2
```

---

### 🔹 2. Create CloudFront Cache Policy

- Name: `HTMLCachingPolicy`
- TTL:
  - Min TTL: `1`
  - Default TTL: `86400`
  - Max TTL: `31536000`
- Headers: `Accept`, `CloudFront-Viewer-Country`, `Host`
- Cookies: `None`
- Query Strings: `All`

---

### 🔹 3. Update CloudFront Behavior

- Behavior: `/*`
- Origin: `wordpress-ec2-origin` (not S3!)
- Cache policy: `HTMLCachingPolicy`
- Origin request policy: `AllViewer`
- TTLs: Use origin cache headers

---

### 🔹 4. Create Lambda@Edge Bypass Function

Region: `us-east-1`

```js
'use strict';
exports.handler = (event, context, callback) => {
  const request = event.Records[0].cf.request;
  const headers = request.headers;

  if (headers.cookie) {
    const cookieStr = headers.cookie.map(x => x.value).join(';');

    if (
      cookieStr.includes('wordpress_logged_in_') ||
      cookieStr.includes('wordpress_sec') ||
      request.uri.startsWith('/wp-admin') ||
      request.uri.startsWith('/cart') ||
      request.uri.startsWith('/checkout')
    ) {
      request.headers['cache-control'] = [{
        key: 'Cache-Control',
        value: 'no-cache'
      }];
    }
  }
  callback(null, request);
};
```

---

### 🔹 5. IAM Trust Policy for Lambda@Edge

In IAM > Role used by Lambda > Trust Relationships:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "lambda.amazonaws.com",
          "edgelambda.amazonaws.com"
        ]
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

---

### 🔹 6. Deploy to Lambda@Edge

From Lambda UI (us-east-1):
- Actions → Deploy to Lambda@Edge
- Distribution: `E3VCMMFN5WXNFI`
- Behavior: `/*`
- Event: `Viewer request`
- Confirm and Deploy ✅

---

## ✅ Testing

### Anonymous user:
```bash
curl -I https://logs.bistrawp.io
```
Expected:
```
X-Cache: Hit from CloudFront
Cache-Control: public, max-age=86400
```

### Logged-in WordPress user:
- Expected: `X-Cache: Miss from CloudFront`

---

## ✅ Final Backups

### EC2:
```bash
aws ec2 create-image --instance-id i-xxxxxxxxxxxx --name "bistrawp-base-ami"
```

### RDS:
- Take snapshot via console

### S3:
- Enable versioning or copy contents to backup bucket

---

You now have a production-grade, plugin-free full edge caching setup running across global CDN with dynamic login-aware rules. 🚀
