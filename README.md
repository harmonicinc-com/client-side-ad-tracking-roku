# Harmonic RAFX SSAI adapter

## Prerequisites
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

## Installation
### ropm (highly recommended)
[ropm](https://github.com/rokucommunity/ropm) package manager is supported. Install it by
```
ropm install @harmonicinc/vos_roku_rafx_ssai
```

### Manual install
Copy the whole `source` folder to your project's `source` folder
Install [roku-requests](https://github.com/rokucommunity/roku-requests) by following the README at the link above

## Usage
1. Create a new player task. It should be launched by another UI logic script.
1. Import required libraries
   ```
   import "pkg:/source/roku_modules/harmonicinc_vos_roku_rafx_ssai/ssai.bs"
   library "Roku_Ads.brs"
   ```
1. Create Harmonic RAFX adapter by 
   ```
   adapter = new harmonic.rafx.ssai.RAFX_SSAI()
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

## Example
Example app is included in `/demo` for reference