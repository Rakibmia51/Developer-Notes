## Backup json data & uploads data from VPS Server MERN



### 1. backend root file create and paste code

```bash
const mongoose = require('mongoose');
const fs = require('fs');
const path = require('path');

// backend-এর .env ফাইল লোড করা
require('dotenv').config();

// .env থেকে বা সরাসরি আপনার MongoDB URI দিন
const MONGO_URI = process.env.MONGO_URI || "mongodb://localhost:27017/erp_jstr_global";

const backupAll = async () => {
    try {
        // ১. ডাটাবেজ কানেক্ট করা
        await mongoose.connect(MONGO_URI);
        console.log('📦 Connected to MongoDB for backup...');

        const backupDir = path.join(__dirname, 'my_backup');
        if (!fs.existsSync(backupDir)) {
            fs.mkdirSync(backupDir, { recursive: true });
        }

        // ২. ডাটাবেজের সমস্ত কালেকশন JSON ফাইলে ব্যাকআপ নেওয়া
        const collections = await mongoose.connection.db.listCollections().toArray();
        for (let col of collections) {
            const colName = col.name;
            const data = await mongoose.connection.db.collection(colName).find({}).toArray();
            
            fs.writeFileSync(
                path.join(backupDir, `${colName}.json`), 
                JSON.stringify(data, null, 2)
            );
            console.log(`✅ Collection backed up: ${colName}`);
        }

        // ৩. uploads ফোল্ডার ব্যাকআপ নেওয়া
        const uploadsDir = path.join(__dirname, 'uploads');
        const backupUploadsDir = path.join(backupDir, 'uploads');

        if (fs.existsSync(uploadsDir)) {
            if (!fs.existsSync(backupUploadsDir)) {
                fs.mkdirSync(backupUploadsDir, { recursive: true });
            }
            fs.cpSync(uploadsDir, backupUploadsDir, { recursive: true });
            console.log('✅ Uploads folder backed up successfully!');
        } else {
            console.log('⚠️ No "uploads" folder found to backup.');
        }

        console.log('🚀 Complete backup (Database JSON + Uploads) finished successfully inside backend/my_backup!');
        process.exit(0);
    } catch (err) {
        console.error('❌ Backup failed:', err);
        process.exit(1);
    }
};

backupAll();

```

Terminal code 
```bash
  node backup-json.js
```


Terminal code 
```bash
apt update && apt install zip -y
```

Terminal code for create zip file
```bash
 zip -r my_backup.zip my_backup
```

local Terminal a paste the code 
লোকাল পিসিতে ডাউনলোড করার সময় জিপ ফাইলের জায়গায় এই ফাইলটি টানবেন:
```bash
scp root@187.77.136.31:/var/www/JSTR-Global-ERP/backend/my_backup.tar.gz "H:\IT\MERN Projects"
```
