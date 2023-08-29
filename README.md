# Harmonic RAFX SSAI adapter

## Installation
### ropm & BrighterScript (highly recommended)
1. [ropm](https://github.com/rokucommunity/ropm) package manager is supported. Install it by
    ```
    ropm install @harmonicinc/vos_roku_rafx_ssai
    ```
1. Create a new player task. It should be launched by another UI logic script.
1. Import required libraries in your BrighterScript file
   ```
   import "pkg:/source/roku_modules/harmonicinc_vos_roku_rafx_ssai/ssai.bs"
   library "Roku_Ads.brs"
   ```

### Manual install
1. Copy source files

   > Choose one below

    **1a. With BrighterScript**

    Copy the whole `source` folder to your project's `source` folder

    Install [roku-requests](https://github.com/rokucommunity/roku-requests) by following the README in that repo

    **1b. With native BrightScript**

    Transpiled BrightScript files are provided in Release as artifact `hlit-rafx-ssai-brs.tar.gz` if you're unable to use BrighterScript. Copy those files in the zip instead of the source files in the repo to your app.

    Dependencies are included, so there's no need to install them separately.

1. Create a new player task. It should be launched by another UI logic script.
1. Import required libraries

   **3a. If you're using BrighterScript**
    
   Import required libraries in your BrighterScript file
   ```
   import "pkg:/source/roku_modules/harmonicinc_vos_roku_rafx_ssai/ssai.bs"
   library "Roku_Ads.brs"
   ```

   **3b. If you're using native BrightScript**

   In the task component XML, import required libraries
   ```
    <script type="text/brightscript" uri="pkg:/source/roku_modules/harmonicinc_vos_roku_rafx_ssai/SSAI.brs" />
    <script type="text/brightscript" uri="pkg:/source/roku_modules/rokurequests_v1/Requests.brs" />
    <script type="text/brightscript" uri="pkg:/source/roku_modules/harmonicinc_vos_roku_rafx_ssai/PodHelper.brs" />
    <script type="text/brightscript" uri="pkg:/source/roku_modules/harmonicinc_vos_roku_rafx_ssai/MetadataParser.brs" />
    <script type="text/brightscript" uri="pkg:/source/roku_modules/harmonicinc_vos_roku_rafx_ssai/DashParser.brs" />
    <script type="text/brightscript" uri="pkg:/source/bslib.brs" />
   ```
    
   Import Roku RAF library in your BrightScript file
   ```
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
1. Check if `result.ssai` is true. If not, the stream is not compatible with the adapter
1. (Optional) Add event listeners. Currently only change on pods will be emitted:
   ```
   adapter.addEventListener(adapter.AdEvent.PODS, podsCallback)

   sub podsCallback(event as object)
       ' Your code here. Get pods by event.adPods
   end sub

   ```
1. Create messsage port and add it to the adapter
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
1. Observe `position` field and create a message loop to feed it to the adapter
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
### Prerequisites
This adapter is written in [BrighterScript](https://github.com/rokucommunity/brighterscript). Install it globally by running
```
npm install brighterscript -g
```
Or install it locally in your Roku app package repository
```
npm install brighterscript --save-dev
```
Transpiling prior to building your Roku app is required.
```
bsc
```

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