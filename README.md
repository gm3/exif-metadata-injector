
```csharp
using System;
using System.IO;
using System.Linq;
using UnityEngine;
using System.Text;
using TMPro;
using UnityEngine.UI;

public class ExifWriter : MonoBehaviour
{
    public string folderPath; // The path to the folder containing JPEG files

    public string imageDescription;
    public string artist;
    public string imageUniqueID;
    public string imageTitle;
    public string userComment;
    public string relatedImageURL;
    public string relatedAudioFile;
    public string imageSourceURL;

    public Button updateButton;

private void Start()
    {
        updateButton.onClick.AddListener(ApplyMetadata);
    }

    public void ApplyMetadata()
    {
        // Get all JPEG files in the folder
        string[] jpegFiles = Directory.GetFiles(folderPath, "*.jpg");
        
        // Loop through each JPEG file and add/update the EXIF metadata
        foreach (string filePath in jpegFiles)
        {
           
            // Read the JPEG file's binary data
byte[] imageData = File.ReadAllBytes(filePath);
if (imageData == null || imageData.Length == 0)
{
    Debug.Log("Failed to read image data from file: " + filePath);
    continue;
}

/// Find the APP1 marker and skip the segment length
        int app1Index = Array.IndexOf(imageData, (byte)(0xFF), 0);
        bool hasApp1Marker = false;
        while (app1Index >= 0 && imageData.Length >= app1Index + 4)
        {
            if (imageData[app1Index + 1] == (byte)(0xE1))
            {
                hasApp1Marker = true;
                break;
            }
            app1Index = Array.IndexOf(imageData, (byte)(0xFF), app1Index + 1);
        }

        // Create the APP1 marker if it doesn't exist
        if (!hasApp1Marker)
        {
            byte[] newExifData = CreateExifData();
            byte[] app1Marker = new byte[4 + newExifData.Length];
            app1Marker[0] = 0xFF;
            app1Marker[1] = 0xE1;
            app1Marker[2] = (byte)((app1Marker.Length >> 8) & 0xFF);
            app1Marker[3] = (byte)(app1Marker.Length & 0xFF);
            Array.Copy(newExifData, 0, app1Marker, 4, newExifData.Length);
            imageData = imageData.Take(2).Concat(app1Marker).Concat(imageData.Skip(2)).ToArray();
            File.WriteAllBytes(filePath, imageData);
        }


            // Find the APP1 marker again and skip the segment length
            app1Index = Array.IndexOf(imageData, (byte)(0xFF), 0);
            while (app1Index >= 0 && imageData.Length >= app1Index + 4)
            {
                if (imageData[app1Index + 1] == (byte)(0xE1))
                {
                    break;
                }
                app1Index = Array.IndexOf(imageData, (byte)(0xFF), app1Index + 1);
            }

            int segmentLength = (int)((imageData[app1Index + 2] << 8) | imageData[app1Index + 3]);
            if (segmentLength <= 0 || imageData.Length < app1Index + 4 + segmentLength)
            {
                Debug.Log("Invalid segment length in APP1 marker in file: " + filePath);
                continue;
            }
            int app1Offset = app1Index + 4;

            // Find the Exif signature and skip the header
            int exifOffset = Array.IndexOf(imageData, (byte)'E', app1Offset);
            if (exifOffset < 0 || imageData.Length < exifOffset + 6 || !Encoding.ASCII.GetString(imageData, exifOffset, 6).Equals("Exif\0\0"))
            {
                Debug.Log("Exif metadata not found in file: " + filePath);
                continue;
            }
            Debug.Log("Exif metadata found in file: " + filePath);
            exifOffset += 6;

            // Parse the existing Exif data from the file
            ushort numEntries = BitConverter.ToUInt16(imageData, app1Offset + exifOffset);
            if (BitConverter.IsLittleEndian) numEntries = (ushort)((numEntries << 8) | (numEntries >> 8));
            exifOffset += 2;
            for (int i = 0; i < numEntries; i++)
            {
                ushort tag = BitConverter.ToUInt16(imageData, app1Offset + exifOffset);
                if (BitConverter.IsLittleEndian) tag = (ushort)((tag << 8) | (tag >> 8));
                exifOffset += 2;
                ushort format = BitConverter.ToUInt16(imageData, app1Offset + exifOffset);
                if (BitConverter.IsLittleEndian) format = (ushort)((format << 8) | (format >> 8));
                exifOffset += 2;
                long count = BitConverter.ToInt32(imageData, app1Offset + exifOffset);
                if (BitConverter.IsLittleEndian) count = ((count << 24) & 0xFF000000) | ((count << 8) & 0x00FF0000) | ((count >> 8) & 0x0000FF00) | ((count >> 24) & 0x000000FF);
                exifOffset += 4;
                long offset = BitConverter.ToInt32(imageData, app1Offset + exifOffset);
                if (BitConverter.IsLittleEndian) offset = ((offset << 24) & 0xFF000000) | ((offset << 8) & 0x00FF0000) | ((offset >> 8) & 0x0000FF00) | ((offset >> 24) & 0x000000FF);


                if (tag == 0x010E && offset + count <= imageData.Length) imageTitle = Encoding.UTF8.GetString(imageData, (int)(app1Offset + offset), (int)count).Trim('\0');
                if (tag == 0x010F && offset + count <= imageData.Length) imageDescription = Encoding.UTF8.GetString(imageData, (int)(app1Offset + offset), (int)count).Trim('\0');
                if (tag == 0x013B && offset + count <= imageData.Length) artist = Encoding.UTF8.GetString(imageData, (int)(app1Offset + offset), (int)count).Trim('\0');
                if (tag == 0xA420 && offset + count <= imageData.Length) imageSourceURL = Encoding.UTF8.GetString(imageData, (int)(app1Offset + offset), (int)count).Trim('\0');
                if (tag == 0x9286 && offset + count <= imageData.Length) userComment = Encoding.UTF8.GetString(imageData, (int)(app1Offset + offset), (int)count).Trim('\0');
                if (tag == 0xA43B && offset + count <= imageData.Length) relatedImageURL = Encoding.UTF8.GetString(imageData, (int)(app1Offset + offset), (int)count).Trim('\0');
                if (tag == 0xA43C && offset + count <= imageData.Length) relatedAudioFile = Encoding.UTF8.GetString(imageData, (int)(app1Offset + offset), (int)count).Trim('\0');


            }

            // Create the new EXIF data as a byte array
            byte[] exifData = CreateExifData();

            // Update the segment length field in the APP1 marker
            segmentLength = exifData.Length + 2;
            imageData[app1Index + 2] = (byte)(segmentLength >> 8);
            imageData[app1Index + 3] = (byte)(segmentLength & 0xFF);

            // Copy the new EXIF data into the file
            Array.Copy(exifData, 0, imageData, app1Offset + exifOffset, exifData.Length);

            // Save the modified binary data back to the file
            File.WriteAllBytes(filePath, imageData);
            Debug.Log("EXIF metadata applied to file: " + filePath);
        }
    }

    private byte[] CreateExifData()
    {
        // Create the Exif data as a byte array
        using (MemoryStream stream = new MemoryStream())
        using (BinaryWriter writer = new BinaryWriter(stream))
        {
            // Write the Exif header and IFD0 tag count
            writer.Write(Encoding.ASCII.GetBytes("Exif\0\0"));
            writer.Write((ushort)0x2A);
            writer.Write((uint)8);

            // Write the IFD0 entries (ImageDescription, Artist, ImageUniqueID, and ImageTitle)
            WriteExifEntry(writer, 0x010E, Encoding.UTF8.GetBytes(imageTitle));
            WriteExifEntry(writer, 0x010F, Encoding.UTF8.GetBytes(imageDescription));
            WriteExifEntry(writer, 0x013B, Encoding.UTF8.GetBytes(artist));

            // Write the Exif IFD offset
            writer.Write((uint)0);

            // Write the Exif data offset and length
            writer.Write(Encoding.ASCII.GetBytes("Exif\0\0"));
            writer.Write((ushort)0);
            writer.Write((ushort)0);
            writer.Write((uint)0x08000000);
            writer.Write((ushort)0x4949);
            writer.Write((ushort)0);

            return stream.ToArray();
        }
    }

    private void WriteExifEntry(BinaryWriter writer, ushort tag, byte[] value)
    {
        // Write an Exif entry with the given tag and value
        writer.Write(tag);
        writer.Write((ushort)2); // Format: ASCII string
        writer.Write((uint)value.Length);
        writer.Write((uint)writer.BaseStream.Position + value.Length + 4);
        writer.Write(value);
        writer.Write((byte)0);
        writer.Write((byte)0);
        writer.Write((byte)0);
        writer.Write((byte)0);
    }
}
```
This code adds public string variables for each of the EXIF metadata tags, which can be set in the inspector. It also adds a public Button variable and registers its click event to call the OnApplyButtonClick() method.

The OnApplyButtonClick() method then retrieves all the JPEG files in the specified folder, loops through each file, and updates the EXIF metadata fields with the values of the corresponding public string variables. Finally, it writes the updated metadata back to each file.

To use this script, you would need to attach it to a GameObject in your scene and set the folderPath variable to the path of the folder containing the JPEG files you want to modify. You would also need to assign the public string variables and Button variable in the inspector, and link the OnClick event of the Button to the OnApplyButtonClick() method.




Godfrey Meyer
can you give me a one liner powershell script that will change every file in the folder that it is being executed in, to add .png to all the files in the folder
Sure! Here's a PowerShell one-liner that will add ".png" to the end of all file names in the current directory:

bash
Copy code
Get-ChildItem | Rename-Item -NewName { $_.Name + '.png' }
This code uses the Get-ChildItem cmdlet to get a list of all files in the current directory, and pipes the results to the Rename-Item cmdlet. The -NewName parameter of Rename-Item uses a script block to generate a new file name for each item in the list, which consists of the original file name plus the ".png" extension.

Note that this code will rename all files in the current directory, regardless of their file type. If you only want to rename specific file types (e.g. ".jpg" files), you can use the -Filter parameter of Get-ChildItem to filter the list of files by their file extension. For example, to rename only ".jpg" files, you can use this command:

sql
Copy code
Get-ChildItem -Filter *.jpg | Rename-Item -NewName { $_.Name + '.png' }



