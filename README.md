```c#
using System;
using System.IO;
using System.Text;
using System.Threading.Tasks;
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Specialized;
using System.Collections.Generic;

class Program
{
    static async Task Main(string[] args)
    {
        string connectionString = "<Your Connection String>";
        string containerName = "<Container Name>";
        string blobName = "<Blob Name>";

        var blobServiceClient = new BlobServiceClient(connectionString);
        var blobContainerClient = blobServiceClient.GetBlobContainerClient(containerName);
        var appendBlobClient = blobContainerClient.GetAppendBlobClient(blobName);

        if (!await appendBlobClient.ExistsAsync())
        {
            await appendBlobClient.CreateAsync();
        }

        int bufferSize = 64 * 1024; // 64 KB
        byte[] buffer = new byte[bufferSize];
        string content = "Hello, this is a test write to an append blob.\n";
        byte[] data = Encoding.UTF8.GetBytes(content);

        long totalBytesWritten = 0;
        int bytesFilled = 0;
        var tasks = new List<Task>();

        while (totalBytesWritten < 12582912) // 12 MB total
        {
            int bytesToCopy = Math.Min(data.Length, bufferSize - bytesFilled);
            Array.Copy(data, 0, buffer, bytesFilled, bytesToCopy);
            bytesFilled += bytesToCopy;

            if (bytesFilled == bufferSize)
            {
                byte[] writeData = new byte[bufferSize];
                Array.Copy(buffer, writeData, bufferSize);
                tasks.Add(AppendBlockAsync(appendBlobClient, writeData));
                totalBytesWritten += bufferSize;
                bytesFilled = 0;
            }
        }

        if (bytesFilled > 0)
        {
            byte[] finalBlock = new byte[bytesFilled];
            Array.Copy(buffer, finalBlock, bytesFilled);
            tasks.Add(AppendBlockAsync(appendBlobClient, finalBlock));
            totalBytesWritten += bytesFilled;
        }

        await Task.WhenAll(tasks);
        Console.WriteLine($"Done. Total Bytes Written: {totalBytesWritten}");
    }

    static async Task AppendBlockAsync(AppendBlobClient appendBlobClient, byte[] data)
    {
        using var ms = new MemoryStream(data);
        await appendBlobClient.AppendBlockAsync(ms);
        Console.WriteLine($"Appended {data.Length} bytes.");
    }
}

```
