# Harmonic RAFX SSAI adapter

## Installation
### Native BrightScript only
1. Download latest release transpiled in BrightScript: [hlit-rafx-ssai-brs.zip](https://github.com/harmonicinc-com/client-side-ad-tracking-roku/releases/latest/download/hlit-rafx-ssai-brs.zip)
    
2. Copy all files inside `/source` to `<your_app_root>/source/hlit-rafx-ssai/`

    Dependencies are included under `/source/roku_modules`, so there's no need to install them separately.

    You should get something like this:
    ```
    └── source
        ├── hlit_rafx_ssai
        │   ├── DashParser.brs
        │   ├── MetadataParser.brs
        │   ├── PodHelper.brs
        │   ├── SSAI.brs
        │   └── roku_modules
        │       ├── bslib
        │       │   └── bslib.brs
        │       └── rokurequests
        │           └── Requests.brs
        └── main.brs
    ```

3. Create a new task component that is responsible for player control & client-side ad tracking. You may use existing ones if you already have one in your app. 
   
   For example, in [Tasks](https://github.com/harmonicinc-com/client-side-ad-tracking-roku/tree/main/demo/components/Tasks) folder in the example app, we have a dedicated `PlayerTask` to handle player-related functions
4. Import required libraries

   In your task component XML, import required libraries.
   ```
    <script type="text/brightscript" uri="pkg:/source/hlit_rafx_ssai/SSAI.brs" />
    <script type="text/brightscript" uri="pkg:/source/hlit_rafx_ssai/PodHelper.brs" />
    <script type="text/brightscript" uri="pkg:/source/hlit_rafx_ssai/MetadataParser.brs" />
    <script type="text/brightscript" uri="pkg:/source/hlit_rafx_ssai/DashParser.brs" />
    <script type="text/brightscript" uri="pkg:/source/hlit_rafx_ssai/roku_modules/rokurequests/Requests.brs" />
    <script type="text/brightscript" uri="pkg:/source/hlit_rafx_ssai/roku_modules/bslib/bslib.brs" />
   ```
   Import Roku RAF library in your BrightScript file
   ```
   library "Roku_Ads.brs"
   ```

### Using ropm & BrighterScript
1. [ropm](https://github.com/rokucommunity/ropm) package manager is supported. Install it by
    ```
    ropm install @harmonicinc/vos_roku_rafx_ssai
    ```
1. Create a new task component that is responsible for player control & client-side ad tracking. You may use existing ones if you already have one in your app. 
   
   For example, in [Tasks](https://github.com/harmonicinc-com/client-side-ad-tracking-roku/tree/main/demo/components/Tasks) folder in the example app, we have a dedicated [PlayerTask.bs](https://github.com/harmonicinc-com/client-side-ad-tracking-roku/blob/main/demo/components/Tasks/PlayerTask.bs) to handle player-related functions
1. Import required libraries in your BrighterScript file
   ```
   import "pkg:/source/roku_modules/harmonicinc_vos_roku_rafx_ssai/ssai.bs"
   library "Roku_Ads.brs"
   ```

## Usage
1. Create Harmonic RAFX adapter by 
   ```
   adapter = new harmonic.rafx.ssai.RAFX_SSAI()

   ' If you're using native BrightScript:
   adapter = harmonicinc_vos_roku_rafx_ssai_RAFX_SSAI()
   ```
1. Provide the stream URL to the adapter
   ```
   result = adapter.getStreamInfo(m.top.video.content.url)
   ```
2. (Optional) Provide the option on whether to send a request first to the session init API to initialise a session.
   
   By default, this is `true` if the options object is not provided. You may set `initRequest` to `false` so that the adapter will obtain the manifest directly. For example:

   ```
   options = {
      initRequest: false
   }
   result = adapter.getStreamInfo(m.top.video.content.url, options)
   ```

   > **_Note_**
   > 
   > By setting the `initRequest` to `true`, you may omit the `sessid` query param in the URL provided to the adapter, and let the SSAI service generate a session ID for you.

3. Check if `result.ssai` is true. If not, the stream is not compatible with the adapter
4. Use the returned stream URL to play the video
   ```
   m.top.video.content.url = result.streamUrl
   ```
   See [PlayerTask](https://github.com/harmonicinc-com/client-side-ad-tracking-roku/blob/main/demo/components/Tasks/PlayerTask.bs#L28) in the example app for reference

   The returned URL is different from the original stream URL. If the returned URL is not used, the SSAI and ad beacons may misalign with the video playback, or beacons will not be fired at all.
5. (Optional) Add event listeners. Currently only change on pods will be emitted:
   ```
   adapter.addEventListener(adapter.AdEvent.PODS, podsCallback)

   sub podsCallback(event as object)
       ' Your code here. Get pods by event.adPods
   end sub

   ```
6. Create message port and add it to the adapter. 
   
   Note that the adapter currently **does not support custom ad beacon firing** at the time of writing. The adapter will handle all the tracking beacons by itself.
   ```
   port = CreateObject("roMessagePort")
   adapter.enableAds({
        player: {
            sgNode: <your video node>,
            port: port
        },
        useStitched: true ' required as firing event on client is not supported yet
    })
   ```
7. Observe `position` field and create a message loop to feed it to the adapter
   ```
   m.top.video.observeFieldScoped("position", port)
    m.top.video.observeFieldScoped("control", port)
    m.top.video.observeFieldScoped("state", port)

    ' Play video
    m.top.video.control = "play"
    
    while true
        msg = wait(1000, port)
        if type(msg) = "roSGNodeEvent" and msg.getField() = "control" and msg.getNode() = m.top.video.id and msg.getData() = "stop" or m.top.video = invalid
            exit while
        end if
        
        curAd = adapter.onMessage(msg)
        if curAd = invalid
            m.top.video.setFocus(true)
        end if

        if "roSGNodeEvent" = type(msg) and "state" = msg.getField() and "finished" = msg.getData() and msg.getNode() = m.top.video.id
            exit while
        end if
    end while

    m.top.video.unobserveFieldScoped("position")
    m.top.video.unobserveFieldScoped("control")
    m.top.video.unobserveFieldScoped("state")
   ```

## Development
### Example app tryout
Example app is included in `/demo` for reference. Demo depends on local SDK instead of the one on npm.

ropm packager is required. Install it globally by
```
npm i ropm -g
```

You'll need to install the dependencies of the SDK first by
```
cd lib
npm i
ropm i
```

Move on to the demo app, do the same by
```
cd ../demo
npm i
ropm i
```

### Build the app
Since the manifest URL is hardcoded, you'll need to replace it prior to building the app.

In `demo/components/Tasks/PlayerTask.bs` line 3, you should have the following:
```
const url = ""
```
Replace it with the manifest URL, for example
```
const url = "https://www.example.com/master.mpd"
```

> **_Optional_**
>
> To force the adapter to obtain the manifest directly instead of using the session init API, set the following in lines 4-6:
> ```
> const options = {
>    initRequest: false
> }
> ```

Then locate to the demo app root directory i.e. `demo/`. Run the following:

```
npm run package
```
The Roku app package will be saved as `/demo/out/demo.zip`. Upload this zip file in Roku debug web UI to install it.

### Develop on Roku w/ live reload
Edit `demo/bsconfig.json`. Fill in the `host` and `password` with the Roku device's IP and developer mode password respectively

Then run the following
```
npm run watch
```

## Appendix

### How the Playback URL and Beaconing URL are Obtained by the Library

> [!NOTE]  
> Applicable when `initRequest` in the options provided is `true` (default is true).

1. The library sends a GET request to the manifest endpoint with the query param "initSession=true". For e.g., a request is sent to:
    ```
    https://my-host/variant/v1/hls/index.m3u8?initSession=true
    ```

2. The ad insertion service (PMM) responds with the URLs. For e.g.,
    ```
    {
        "manifestUrl": "./index.m3u8?sessid=a700d638-a4e8-49cd-b288-6809bd35a3ed&vosad_inst_id=pmm-0",
        "trackingUrl": "./metadata?sessid=a700d638-a4e8-49cd-b288-6809bd35a3ed&vosad_inst_id=pmm-0"
    }
    ```

3. The library constructs the URLs by combining the host and base path in the original URL and the relative URLs obtained. For e.g.,
    ```
    Manifest URL: https://my-host/variant/v1/hls/index.m3u8?sessid=a700d638-a4e8-49cd-b288-6809bd35a3ed&vosad_inst_id=pmm-0

    Metadata URL: https://my-host/variant/v1/hls/metadata?sessid=a700d638-a4e8-49cd-b288-6809bd35a3ed&vosad_inst_id=pmm-0
    ```

> [!NOTE]  
> The resulting manifest URL above can be obtained in the returned result's `streamUrl` when calling `getStreamInfo`.
