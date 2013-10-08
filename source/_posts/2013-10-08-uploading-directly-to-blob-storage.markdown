---
layout: post
title: "super-awesome"
date: 2013-10-08 14:52
comments: true
categories: [JavaScript, Windows Azure]
---

## Problem
[Firstonsite](http://firstonsite.se)'s goal is to be a marketplace for news media, allowing anyone to upload images and video of interesting events and offer it news desks for sale. To be able to handle large amounts of media files from many concurrent users we wanted to leverage the power of [Windows Azure](http://www.windowsazure.com/sv-se/) and its Blob Storage to allow the users to upload directly into our blob storage without first having to pass through our web back-end. This allow us to avoid a bottleneck when several users wants to upload large files and remove the need for creating several web roles just for this purpose.

To accomplish this we needed the new new [File API](http://www.w3.org/TR/FileAPI/) that comes in HTML 5 that allows JavaScript to handle file upload without having to touch the backend. The Firstonsite project have previously decided to require the user to have a HTML 5 compliant browser, using the &lt;video&gt; tag for instance, so this was not a problem.

## Solution
When we first tried to implement this solution we ran into an unexpected problem, Blob Storage does not have [CORS](http://en.wikipedia.org/wiki/Cross-origin_resource_sharing) support ([yet](http://feedback.windowsazure.com/forums/217298-storage/suggestions/2850796-support-cross-origin-resource-sharing-cors-via-a?page=1&per_page=20)).

We decided to push the feature until CORS was supported but recently I found a very interesting [blog by Gaurav Mantri](http://gauravmantri.com/2013/02/16/uploading-large-files-in-windows-azure-blob-storage-using-shared-access-signature-html-and-javascript/) with a workaround of the problem. The workaround is to place the code for the file upload on the Blob Storage thus removing the need for cross domain access.

## Example code
I have created an example site implementing Mantri's solution using a controller to create a [Shared Access Signature](http://msdn.microsoft.com/en-us/library/windowsazure/jj721951.aspx) and sending this to the static HTML page on the Blob Storage. In order to prevent people from draining my Azure account dry I used the MVC 4 Simple Membership provider template to add login to the page.

The upload controller creates the SAS and adds it as a query parameter to the address of the static site. To avoid Blob Storage parsing the parameter (assuming that it is a SAS used to access the requested HTML file) the string is base 64 encoded first.

The upload.js script running at the Blob Storage extracts the sas query parameter using [purl.js](https://github.com/allmarkedup/purl) and decodes the string to get the URI for the file to be uploaded to. 

The user can select a file and when the **Upload File** button is pressed a FileReader starts reading slices of that file. Each slice is uploaded to the Blob Storage together with a six digit block id. This Id is later used to combine the slices at the Blob Storage. 

If an error occurs the code will retry the transmission of the current slice several times before failing the file upload. This takes care of occasional glitches that can occur normally. 

After all slices have been uploaded a list of all slices and their order are sent as a block list and the file transfer will be complete.

All code can be found at https://github.com/MartinWa/BlobUploader

## How to test it
The easiest way to test this is to clone the code from GitHub to a local folder (using git), open the solution file in Visual Sudio 2012 (with the Azure SDK), start the storage emulator and run the debugger.

The contents of the blobstorageFiles folder must be stored on the development Blob Storage in a container named public (with public access rights). Note that the content type of the HTML file must be set to text/html. I do all this using the [Azure Storage Explorer tool](http://azurestorageexplorer.codeplex.com/). If you have done everything correct you should be able to browse to http://127.0.0.1:10000/devstoreaccount1/public/upload.html.

In order to be able to access the Upload tab you must create a uploader entry in the webpages_Roles table and then set that role to your user in webpages_UsersInRoles.

## Deploy your own
If you like to deploy to your own Windows Azure Web Site clone the repository to your own GitHub account.
Create a new Azure Web Site using the custom create wizard
![Custom Create Web Site](https://dynablog.s3.amazonaws.com/uploads/article_image/image/10/CreateCustomWebsite.png)

adding your GitHub repository 
![Integrate GitHub](https://dynablog.s3.amazonaws.com/uploads/article_image/image/11/IntegrateGitHub.png)

Now the management portal will deploy the main branch of the repository to the Web Site. It will also deploy any changes you push to main giving you an instant continuous deployment.

Since we need file storage we also need to create a storage account
![Create Storage Account](https://dynablog.s3.amazonaws.com/uploads/article_image/image/12/CreateStorage.png)

Now all that remains is to set the database connection and the storage account. Go to configure on the Web Site and add an app setting:
<code>StorageConnection : DefaultEndpointsProtocol=https;AccountName=xxx;AccountKey=***</code> replacing the account name with the name of your storage account and *** with the primary access key. Then rename the **DefaultConnection** connection string to **DatabaseConnection**

The site should now be up. As in the local case you need to add roles and copy the static HTML page to the Blob Storage (set up the account in the Windows Azure explorer).

See my site at http://blobupload.azurewebsites.net/

## Future improvements
The file upload should be done in parallel to improve upload speed..