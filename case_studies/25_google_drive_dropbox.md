# NEW Case Study: Google Drive / Dropbox
- **Requirements:** Sync files across devices, support large files (50GB+), minimize bandwidth.
- **Architecture:** Metadata DB (SQL) separate from Block Storage (S3).
- **Key Insight:** Do not upload whole files. Break files into 4MB chunks (Blocks). Calculate hashes. Only upload blocks whose hashes don't exist on the server (Delta Sync / Deduplication).
