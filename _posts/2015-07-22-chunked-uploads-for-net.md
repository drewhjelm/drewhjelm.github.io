---
layout: post
title: Chunked uploads for .NET
---

I recently ran into a problem with HTTP timeouts and the default ASP.NET FileUpload control. I was able to find a solution by implementing plUpload as demonstrated by Rick Strahl’s fantastic [plUploadHandler](http://weblog.west-wind.com/posts/2013/Mar/12/Using-plUpload-to-upload-Files-with-ASPNET). The only challenge is that I’ve found antivirus with web applications can be overzealous and slow down the uploads for large files. Increasing the chunkStream copy size from the code’s default 16384 to the .NET large object heap max of 81920 (largest multiple of 4096 that fits into the 85k large object heap) improved performance, as did uploading larger chunks.