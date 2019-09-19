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
Here is the contents of my service worker.

```


console.log('[ServiceWorker] hello from the service worker');
var buildNumber = 1;
var cacheName = 'mycache';
var possible403Extensions =
{
    ".dll": "application/octet-stream",
    ".wasm": "application/wasm"
};
var debug403 = false;

// https://stackoverflow.com/questions/16245767/creating-a-blob-from-a-base64-string-in-javascript
function b64toBlob(b64Data, contentType, sliceSize) {
    contentType = contentType || '';
    sliceSize = sliceSize || 512;

    var byteCharacters = atob(b64Data);
    //console.log("b64toBlob: b64 len = " + b64Data.length + ', byteCharacters lenth = ' + byteCharacters.length);
    var byteArrays = [];

    for (var offset = 0; offset < byteCharacters.length; offset += sliceSize) {
        var slice = byteCharacters.slice(offset, offset + sliceSize);

        var byteNumbers = new Array(slice.length);
        for (var i = 0; i < slice.length; i++) {
            byteNumbers[i] = slice.charCodeAt(i);
        }

        var byteArray = new Uint8Array(byteNumbers);

        byteArrays.push(byteArray);
    }
    //console.log("b64ToBlob: byteArrays length = " + byteArrays.length);
    var blob = new Blob(byteArrays, { type: contentType });
    return blob;
}

console.log('[ServiceWorker] Adding Install Listener');
self.addEventListener('install', function (e) {
    console.log('[ServiceWorker] Installing');
    e.waitUntil(self.skipWaiting());
});

console.log('[ServiceWorker] Adding Activate Listener');
self.addEventListener('activate', event => {
    console.log('[ServiceWorker] activate: claiming start');
    event.waitUntil(self.clients.claim());
    console.log('[ServiceWorker] activate: claiming done');
});

function cacheResponse(response, event)
{
    // IMPORTANT: Clone the response. A response is a stream
    // and because we want the browser to consume the response
    // as well as the cache consuming the response, we need
    // to clone it so we have two streams.
    var responseToCache = response.clone();
    event.EndTime = new Date();
    caches.open(cacheName)
        .then(function (cache) {
            if (cache) {
                //console.log(event.request.url + ' cache object was valid. saving. Time = ' + (event.EndTime - event.StartTime) + ' mSecs');
                cache.put(event.request.url, responseToCache);
            }
            else {
                console.log(event.request.url + ' cache object was NOT valid. not saved, Time = ' + (event.EndTime - event.StartTime) + ' mSecs');
            }
        });
}

// this version works!
console.log('[ServiceWorker] Adding Fetch Listener');
self.addEventListener('fetch', function (event) {
    //console.log('[ServiceWorker] fetching ' + event.request.url, event);
    event.StartTime = new Date();
    var url = new URL(event.request.url);
    //console.log('[ServiceWorker] url pathname ' + url.pathname, url);
    if (url.pathname === "/")
    {
        event.EndTime = new Date();
        //console.log(event.request.url + ' : detected root and returning nothing. fetch time = ' + (event.EndTime - event.StartTime) + ' mSecs');
        return;
    }
    
    event.respondWith(
        // look in the cache
        caches.match(event.request.url)
            .then(function (response) {
                // was the match successful?
                //console.log(event.request.url + ' matched response ', response);
                if (response) {
                    event.EndTime = new Date();
                    //console.log(event.request.url + ' returning result from cache in ' + (event.EndTime - event.StartTime) + ' mSecs', response);
                    return response;
                }
                else
                { 
                    //console.log(event.request.url + ' not found in cache. fetching from server');
                }

                return fetch(event.request).then(
                    function (response) {
                        // did we get a 403?
                        if (debug403 === true || (response !== null && response.status === 403))
                        {
                            console.log(event.request.url + ' response of 403 detected');

                            // yes. is this a file we can try to handle?
                            var extension = url.pathname.substring(url.pathname.lastIndexOf(".")).toLowerCase();
                            console.log(event.request.url + ': extension = ' + extension);
                            if (extension in possible403Extensions)
                            {
                                console.log(event.request.url + ': this is an extension we can handle. getting the txt version of the file');
                                var newExtension = extension.replace(".", "-") + ".txt";
                                var textUrl = url.href.replace(extension, newExtension);
                                console.log(event.request.url + ': retrieving txt version of the file ' + textUrl);
                                var textFileStart = new Date();
                                var newResponse = fetch(textUrl).then(function (textFileResponse) {
                                    var textFileEnd = new Date();
                                    console.log(textUrl + ': 1 fetched in ' + (textFileEnd - textFileStart) + ' mSec', textFileResponse);
                                    var newResponse = textFileResponse.text().then(function (text) {
                                        //console.log(textUrl + ': 3 found text with length of ' + text.length, text);
                                        var contentType = possible403Extensions[extension];
                                        console.log(textUrl + ': content-type = ', contentType);
                                        var blob = b64toBlob(text, contentType);
                                        console.log(textUrl + ': 3.5 blob created with length of ' + blob.size, blob);
                                        var init = { "status": 200, "statusText": "OK" };
                                        // https://developer.mozilla.org/en-US/docs/Web/API/Response/Response
                                        var newResponse = new Response(blob, init);
                                        console.log(textUrl + ': 004 new response ', newResponse);

                                        // cache the file
                                        cacheResponse(newResponse, event);

                                        // return it
                                        return newResponse;
                                    });

                                    return newResponse;
                                });

                                console.log(event.request.url + ': 555 overriding response ', newResponse);

                                // return the response 
                                return newResponse;
                            }
                            else
                            {
                                console.log(event.request.url + ': this is not an extension we can handle');
                            }
                        }
                        else
                        {
                            // console.log('it was not a 403');
                        }

                        // Check if we received a valid response
                        if (!response || response.status !== 200 || response.type !== 'basic') {
                            event.EndTime = new Date();
                            console.log(event.request.url + ' fetch response was not valid. not saving to cache. response is null =' + (response === null) + ', response.status = ' + response.status + ', response.type = ' + response.type + ', fetch time = ' + (event.EndTime - event.StartTime) + ' mSecs');
                            return response;
                        }
                        else
                        {
                            //console.log(event.request.url + ' fetch response is valid. Saving to cache');
                        }

                        cacheResponse(response, event);
                        return response;
                    }
                );
            })
    );
});

```

The critical parts to get it to handle requests BEFORE blazor starts requesting the files is the ***self.skipWaiting()*** line below.
```
console.log('[ServiceWorker] Adding Install Listener');
self.addEventListener('install', function (e) {
    console.log('[ServiceWorker] Installing');
    e.waitUntil(self.skipWaiting());
});

console.log('[ServiceWorker] Adding Activate Listener');
self.addEventListener('activate', event => {
    console.log('[ServiceWorker] activate: claiming start');
    event.waitUntil(self.clients.claim());
    console.log('[ServiceWorker] activate: claiming done');
});
```

The service worker is in its current raw 'not for public consumption' form. My goal in sharing it was to provide a starting point for others to build their own research on. 

Please share with me any thoughts you have or improvements that could be made. Together we can come up with a solution that will get us by until a more polished supported version becomes available from Microsoft. 

I welcome all comments!
Joe
