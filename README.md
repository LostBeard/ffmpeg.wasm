<p align="center">
  <a href="#">
    <img alt="ffmpeg.wasm" width="128px" height="128px" src="https://github.com/LostBeard/ffmpeg.wasm/blob/main/apps/website/static/img/logo192.png"></img>
  </a>
</p>

# Fork of the awesome project [ffmpeg.wasm](https://github.com/ffmpegwasm/ffmpeg.wasm)

Forked to test customization of ffmpeg-core including adding [WORKERFS](https://emscripten.org/docs/api_reference/Filesystem-API.html#filesystem-api-workerfs) filesystem support. ✅

This fork is used by [SpawnDev.BlazorJS.FFmpegWasm](https://github.com/LostBeard/SpawnDev.BlazorJS.FFmpegWasm) to bring ffmpeg.wasm into Blazor WASM apps.

## Demo code

index.html  
```html
<html>
<head>
    <script src="transcode.js"></script>
</head>
<body>
    <p>Test using <a href="https://github.com/LostBeard/ffmpeg.wasm">ffmpeg.wasm v0.12.7</a> a fork of the awesome project at <a href="https://github.com/ffmpegwasm/ffmpeg.wasm">ffmpeg.wasm</a></p>
    <video autoplay muted id="video-result" controls></video><br />
    <button disabled id="load-button">Load ffmpeg-core (~31 MB)</button><br />
    <button disabled id="transcode-button">Transcode remote file</button><br />
    <input style="width: 100%;" disabled id="remote-file" type="text" value="https://raw.githubusercontent.com/ffmpegwasm/testdata/master/Big_Buck_Bunny_180_10s.webm"><br />
    <p id="log-div"></p>
    <p>Open Developer Tools (Ctrl+Shift+I) to View Logs</p>
</body>
</html>
```

transcode.js  
```js
"use strict";

const useWorkerFSIfAvailable = true;
const useMultiThreadIfAvailable = true;
const baseURLFFMPEG = 'ffmpeg-wasm/ffmpeg';
const baseURLCore = 'ffmpeg-wasm/core';
const baseURLCoreMT = 'ffmpeg-wasm/core-mt';
const workerBFSLoaderURL = 'worker.loader.js';
var ffmpeg = null;
var loadBtn = null;
var transcodeBtn = null;
var logDiv = null;
var videoEl = null;
var remoteFileInput = null;

const toBlobURL = async (url, mimeType) => {
    var resp = await fetch(url);
    var body = await resp.blob();
    var blob = new Blob([body], { type: mimeType });
    return URL.createObjectURL(blob);
};

const fetchFile = async (url) => {
    var resp = await fetch(url);
    var buffer = await resp.arrayBuffer();
    return new Uint8Array(buffer);
};

const load = async () => {
    loadBtn.setAttribute('disabled', true);
    const ffmpegBlobURL = await toBlobURL(`${baseURLFFMPEG}/ffmpeg.js`, 'text/javascript');
    await import(ffmpegBlobURL);
    ffmpeg = new FFmpegWASM.FFmpeg();
    ffmpeg.on('log', ({ message }) => {
        logDiv.innerHTML = message;
        console.log(message);
    });
    // check if SharedArrayBuffer is supported via crossOriginIsolated global var
    // https://developer.mozilla.org/en-US/docs/Web/API/crossOriginIsolated
    // multi-threading may not work with all commands
    if (useMultiThreadIfAvailable && window.crossOriginIsolated) {
        transcodeBtn.innerHTML = 'Transcode webm to mp4 (multi-threaded)';
        await ffmpeg.load({
            workerLoadURL: await toBlobURL(`${baseURLFFMPEG}/814.ffmpeg.js`, 'text/javascript'),
            coreURL: await toBlobURL(`${baseURLCoreMT}/ffmpeg-core.js`, 'text/javascript'),
            wasmURL: await toBlobURL(`${baseURLCoreMT}/ffmpeg-core.wasm`, 'application/wasm'),
            workerURL: await toBlobURL(`${baseURLCoreMT}/ffmpeg-core.worker.js`, 'application/javascript'),
        });
    } else {
        transcodeBtn.innerHTML = 'Transcode webm to mp4 (single-threaded)';
        await ffmpeg.load({
            workerLoadURL: await toBlobURL(`${baseURLFFMPEG}/814.ffmpeg.js`, 'text/javascript'),
            coreURL: await toBlobURL(`${baseURLCore}/ffmpeg-core.js`, 'text/javascript'),
            wasmURL: await toBlobURL(`${baseURLCore}/ffmpeg-core.wasm`, 'application/wasm'),
        });
    }
    console.log('ffmpeg.load success');
    transcodeBtn.removeAttribute('disabled');
    remoteFileInput.removeAttribute('disabled');
}

const transcodeRemoteFile = async () => {
    transcodeBtn.setAttribute('disabled', true);
    await ffmpeg.writeFile('input.webm', await fetchFile('https://raw.githubusercontent.com/ffmpegwasm/testdata/master/Big_Buck_Bunny_180_10s.webm'));
    await ffmpeg.exec(['-i', 'input.webm', 'output.mp4']);
    const outputUint8Array = await ffmpeg.readFile('output.mp4');
    videoEl.src = URL.createObjectURL(new Blob([ outputUint8Array ], { type: 'video/mp4' }));
    transcodeBtn.removeAttribute('disabled');
}

addEventListener("load", async (event) => {
    remoteFileInput = document.querySelector('#remote-file');
    loadBtn = document.querySelector('#load-button');
    loadBtn.addEventListener('click', async () => await load());
    loadBtn.removeAttribute('disabled');
    transcodeBtn = document.querySelector('#transcode-button');
    transcodeBtn.addEventListener('click', async () => await transcodeRemoteFile());
    logDiv = document.querySelector('#log-div');
    videoEl = document.querySelector('#video-result');
    console.log('window loaded');
});

```