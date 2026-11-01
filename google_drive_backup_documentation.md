# Automated & Manual MERN Stack Backup to Google Drive (`JSTR-Global-ERP`)

A robust, production-grade automated and manual backup solution for MERN stack applications (`JSTR-Global-ERP`). This script exports all MongoDB database collections into JSON files, copies local media storage (`uploads/`), compresses the entire payload into a `.zip` archive, uploads it directly to Google Drive via OAuth2, and cleans up local temporary files.

---

## 📌 Features

- **Database Dump:** Connects via Mongoose and exports all collections as formatted JSON files.
- **Media Preservation:** Copies local asset folders (`uploads/`).
- **In-Memory Compression:** Uses `adm-zip` for cross-platform fast `.zip` archive generation.
- **Quota-Bypassing Drive Upload:** Utilizes OAuth2 Refresh Tokens with personal Google Account storage, bypassing Service Account storage quota limitations (`403 storageQuotaExceeded`).
- **Dual Execution Mode:**
  - **Manual Mode (`--now`):** Immediate on-demand execution for instant backups.
  - **Automated Mode (`node-cron`):** Daily background task scheduled via cron patterns.
- **Automatic Storage Cleanup:** Deletes temporary JSON dumps and zip files locally after a successful cloud upload.

---

## 🛠️ Prerequisites & Installation

Ensure Node.js and MongoDB are installed on your environment.

1. Clone or copy the repository files.
2. Install the necessary dependencies:

```bash
npm install mongoose googleapis adm-zip dotenv node-cron
```

---

## 🔑 Google Cloud Console & OAuth2 Setup Guide

To avoid Google Service Account quota restrictions, this setup uses an **OAuth2 Refresh Token** linked to your personal Gmail Google Drive storage.

### Step 1: Create Credentials in Google Cloud Console
1. Navigate to the [Google Cloud Console](https://console.cloud.google.com/).
2. Create a new project or select an existing one.
3. Go to **APIs & Services > Library**, search for **Google Drive API**, and click **Enable**.
4. Go to **APIs & Services > Credentials** and click **+ CREATE CREDENTIALS > OAuth client ID**.
5. Select **Web application** as the application type.
6. Under **Authorized redirect URIs**, add both:
   - `https://developers.google.com/oauthplayground`
   - `https://developers.google.com/oauthplayground/`
7. Click **Create** and save your **Client ID** and **Client Secret**.

### Step 2: Generate Refresh Token via OAuth2 Playground
1. Open [OAuth 2.0 Playground](https://developers.google.com/oauthplayground/).
2. Click the **Gear Icon (⚙️)** in the top-right corner.
3. Check **Use your own OAuth credentials**.
4. Input your **Client ID** and **Client Secret**.
5. Under **Step 1 (Select & authorize APIs)**, scroll to **Drive API v3** and select:
   - `https://www.googleapis.com/auth/drive.file`
6. Click **Authorize APIs** and log in with your target Google Account.
7. In **Step 2 (Exchange authorization code for tokens)**, click **Exchange authorization code for tokens**.
8. Copy the generated **`refresh_token`** value from the response payload.

---

## ⚙️ Environment Variables Configuration (`.env`)

Create a `.env` file in the root backend directory:

```env
MONGO_URI=mongodb://localhost:27017/erp_jstr_global
GOOGLE_DRIVE_FOLDER_ID=1GqfB8STrl7odqrNCj7lnZSBXZJRlJRNr
GOOGLE_CLIENT_ID=your_google_client_id_here
GOOGLE_CLIENT_SECRET=your_google_client_secret_here
GOOGLE_REFRESH_TOKEN=your_google_refresh_token_here
```

---

## 📜 Full Backup Source Code (`backup-json.js`)

Save the following code inside `backup-json.js`:

```javascript
const mongoose = require('mongoose');
const fs = require('fs');
const path = require('path');
const { google } = require('googleapis');
const AdmZip = require('adm-zip');
const cron = require('node-cron');

require('dotenv').config();

const MONGO_URI = process.env.MONGO_URI || "mongodb://localhost:27017/erp_jstr_global";
const FOLDER_ID = process.env.GOOGLE_DRIVE_FOLDER_ID || '1GqfB8STrl7odqrNCj7lnZSBXZJRlJRNr';

const CLIENT_ID = process.env.GOOGLE_CLIENT_ID;
const CLIENT_SECRET = process.env.GOOGLE_CLIENT_SECRET;
const REDIRECT_URI = 'https://developers.google.com/oauthplayground';
const REFRESH_TOKEN = process.env.GOOGLE_REFRESH_TOKEN;

// 1. Google Drive Upload Service via OAuth2
const uploadToGoogleDrive = async (filePath, fileName) => {
    try {
        const oAuth2Client = new google.auth.OAuth2(
            CLIENT_ID,
            CLIENT_SECRET,
            REDIRECT_URI
        );

        oAuth2Client.setCredentials({ refresh_token: REFRESH_TOKEN });
        const drive = google.drive({ version: 'v3', auth: oAuth2Client });

        const fileMetadata = {
            name: fileName,
            parents: [FOLDER_ID],
        };

        const media = {
            mimeType: 'application/zip',
            body: fs.createReadStream(filePath),
        };

        const response = await drive.files.create({
            requestBody: fileMetadata,
            media: media,
            fields: 'id, name, webViewLink',
        });

        console.log(`🚀 Backup successfully uploaded to Google Drive! File ID: ${response.data.id}`);
        console.log(`🔗 File Link: ${response.data.webViewLink}`);
    } catch (err) {
        console.error('❌ Google Drive upload failed:', err.message);
    }
};

// 2. Core Backup and Archiving Core Process
const runBackupAndDriveUpload = async () => {
    try {
        if (mongoose.connection.readyState !== 1) {
            await mongoose.connect(MONGO_URI);
            console.log('📦 Connected to MongoDB...');
        }

        const backupDir = path.join(__dirname, 'my_backup');
        if (!fs.existsSync(backupDir)) fs.mkdirSync(backupDir, { recursive: true });

        // Step A: Dump MongoDB collections to JSON
        const collections = await mongoose.connection.db.listCollections().toArray();
        for (let col of collections) {
            const data = await mongoose.connection.db.collection(col.name).find({}).toArray();
            fs.writeFileSync(path.join(backupDir, `${col.name}.json`), JSON.stringify(data, null, 2));
        }

        // Step B: Copy local uploads media folder
        const uploadsDir = path.join(__dirname, 'uploads');
        if (fs.existsSync(uploadsDir)) {
            fs.cpSync(uploadsDir, path.join(backupDir, 'uploads'), { recursive: true });
        }

        // Step C: Compress backup folder using adm-zip
        const zipFileName = `backup-${Date.now()}.zip`;
        const zipFilePath = path.join(__dirname, zipFileName);
        console.log('🗜️ Compressing backup using Node.js...');
        
        const zip = new AdmZip();
        zip.addLocalFolder(backupDir, 'my_backup');
        zip.writeZip(zipFilePath);
        console.log('✅ Zip file created successfully.');

        // Step D: Upload to Google Drive
        console.log('☁️ Uploading to Google Drive...');
        await uploadToGoogleDrive(zipFilePath, zipFileName);

        // Step E: Clean up temporary files
        fs.rmSync(backupDir, { recursive: true, force: true });
        fs.unlinkSync(zipFilePath);
        console.log('🧹 Local temporary backup files cleaned up successfully.');

    } catch (err) {
        console.error('❌ Backup Process failed:', err);
    }
};

// ==========================================
// Dual Mode Execution Logic
// ==========================================

const isManualRun = process.argv.includes('--now');

if (isManualRun) {
    // Manual Execution
    console.log('⚡ Manual backup initiated...');
    runBackupAndDriveUpload().then(() => {
        console.log('🎉 Manual backup complete.');
        process.exit(0);
    });
} else {
    // Automated Cron Execution (Default: Daily at 03:00 AM)
    console.log('⏳ Scheduled backup service started.');
    console.log('⏰ Backup scheduled daily at 03:00 AM.');

    cron.schedule('0 3 * * *', async () => {
        console.log('⏰ Starting daily automated backup execution...');
        await runBackupAndDriveUpload();
    });
}
```

---

## 🚀 Execution & Usage

### 1. Manual Immediate Run
To execute a backup instantly on-demand:
```bash
node backup-json.js --now
```

### 2. Production Service Setup via PM2
To keep the automated cron scheduler running persistently in the background on VPS servers:

```bash
# Install PM2 globally (if not installed)
npm install -g pm2

# Start the backup service
pm2 start backup-json.js --name "erp-auto-backup"

# Configure PM2 autostart on system reboot
pm2 startup
pm2 save

# Inspect backup output logs
pm2 logs erp-auto-backup
```

---

## 🛡️ Security Best Practices (`.gitignore`)

Ensure your secrets are never exposed on public repositories. Add these lines to your `.gitignore` file:

```gitignore
node_modules/
.env
my_backup/
*.zip
backup.log
```