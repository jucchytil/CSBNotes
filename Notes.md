# Hello Everyone
Sorry for the imperfect distribution of this info. For now, here is the key things I have learned.


# Problem
Blazor client side DLLs are being block at a firewall. The browser gets a 403 and the app does not load. 


# Solution
Use a service worker to detect the 403 and then download a base64 encoded version of the DLL being blocked, convert it back to a DLL, save the DLL in the browser cache and pass the DLL back to the Blazor.  When the page is refreshed, it will find the DLL in the cache and start up quickly. 


# Steps
* Create new text files on the server that contain Base64 versions of the DLLs
* Configure index.html to register the service worker
* Let the service worker load immediately and begin handling the 403s and managing the files

## Creating Base64 Files

In my Azure DevOps pipeline, I have PowerShell that converts the DLLs to Base64 files. Nothing fancy here.

    # get the bytes from the binary file (wasm or dll) 
    $Bytes            = [System.IO.File]::ReadAllBytes($fileInfo.FullName)

    # convert the bytes to a base64 string
    $EncodedData      = [Convert]::ToBase64String($Bytes)

    # create an extension change (ex: .dll becomes -dll.txt)
    $extensionChanges = "{0}{1}" -f $fileInfo.Extension.Replace(".","-"), ".txt"

    # create a new file name based on the existing one but change its extension (ex: System.Text.Json.dll -> System.Text.Json-dll.txt)
    $encodedFileName  = $file.Replace($fileinfo.Extension, $extensionChanges)

    # save the Base64 content to the new file
    add-content -path $encodedFileName $EncodedData

Be sure to configure your delivery system to use a content-type of *text/plain* when the text files are downloaded. So far, I have not had trouble with this but we'll see if I need to change this down the road.

## Configure Index.html
****Disclaimer:**** I am **really** new to Service Workers and have a LOT to learn. If you have wisdom to share, I am ALL EARS!

All documentation I've read so far seem to want service workers to load at the end of the index.html. I initially had a problem with this as Blazor started trying to download the DLLs before the service worker was fully loaded and the first-use experience was terrible. I did not want to tell users to refresh their browser to get it to work.  

I tried a couple things to fix this.
* Loading the service worker at the top of index.html
* configuring the service worker to 'load immediately'

Here is the header portion of my index.html. The rest of the index.html is just standard client side blazor stuff. 
```
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width" />
  <title>MyApp</title>
  <base href="/" />
  <link rel="script" href="ServiceWorker.js" />
  <script>
     if ('serviceWorker' in navigator) 
     {
        console.log('Registering service worker');
        navigator.serviceWorker.register('/ServiceWorker.js')
               .then(function (registration) 
               {
                  console.log('Service Worker Registration succeeded');
               });
     }
     else 
     {
       console.log('Service worker already registered');
     }
  </script>
  <link rel="stylesheet" href="lib/bootstrap/dist/css/bootstrap.min.css" />
  <link rel="stylesheet" href="lib/font-awesome/css/font-awesome.min.css" />
  <link rel="stylesheet" href="css/animate.css" />
  <link rel="stylesheet" href="css/style.css" />
  <link rel="shortcut icon" type="image/x-icon" href="favicon.ico" />
  <script src="js/localStorage.js"></script>
</head>
```

I put a link to ServiceWorker.js file immediately after the base tag.
  <link rel="script" href="ServiceWorker.js" />

I then register the in the following script tag.  Thats it.

## ServiceWorker.js

