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
    <p>Test using <a href="https://github.com/LostBeard/ffmpeg.wasm">ffmpeg.wasm custom v0.12.7</a>. A fork of the
        awesome project at <a href="https://github.com/ffmpegwasm/ffmpeg.wasm">ffmpeg.wasm</a></p>
    <div style="padding: 5px;">
        <button disabled id="load-button">Load ffmpeg-core (~31 MB)</button><br />
        <video autoplay muted id="video-result" controls></video><br />
    </div>
    <div style="padding: 5px;">
        Local file input:<br />
        <input disabled type="file" id="local-file" accept=".mp4,.m4v,.webm,.avi,.mkv,.mov,.wmv" /><br />
    </div>
    <div style="padding: 5px;">
        Remote file input:<br />
        <input style="width: 100%;" disabled id="remote-file" type="text"
            value="https://raw.githubusercontent.com/ffmpegwasm/testdata/master/Big_Buck_Bunny_180_10s.webm"><br />
        <button disabled id="transcode-button">Transcode remote file</button><br />
    </div>
    <p id="log-div"></p>
    <p>Open Developer Tools (Ctrl+Shift+I) to View Logs</p>
</body>
</html>
```

transcode.js  
```js  
"use strict";

const useWorkerFSIfAvailable = false;
const useMultiThreadIfAvailable = true;
// uses custom build of ffmpeg.wasm v0.12.6 with pull requests #562 and #581 implemented; found here
// https://github.com/LostBeard/ffmpeg.wasm/releases/download/v0.12.7/LostBeard-ffmpeg.wasm.v0.12.7.zip
const baseURLFFMPEG = 'ffmpeg-wasm/ffmpeg';
const baseURLCore = 'ffmpeg-wasm/core';
const baseURLCoreMT = 'ffmpeg-wasm/core-mt';
var ffmpeg = null;
var loadBtn = null;
var transcodeBtn = null;
var logDiv = null;
var videoEl = null;
var localFileInput = null;
var remoteFileInput = null;

const toBlobURL = async (url, mimeType) => {
    var resp = await fetch(url);
    var body = await resp.blob();
    var blob = new Blob([body], { type: mimeType });
    return URL.createObjectURL(blob);
};

const fetchFile = async (url) => {
    var resp = typeof url === 'string' ? await fetch(url) : url;
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
    if (useMultiThreadIfAvailable && window.crossOriginIsolated) {
        transcodeBtn.innerHTML = 'Transcode remote file (multi-threaded)';
        await ffmpeg.load({
            workerLoadURL: await toBlobURL(`${baseURLFFMPEG}/814.ffmpeg.js`, 'text/javascript'),
            coreURL: await toBlobURL(`${baseURLCoreMT}/ffmpeg-core.js`, 'text/javascript'),
            wasmURL: await toBlobURL(`${baseURLCoreMT}/ffmpeg-core.wasm`, 'application/wasm'),
            workerURL: await toBlobURL(`${baseURLCoreMT}/ffmpeg-core.worker.js`, 'application/javascript'),
        });
    } else {
        transcodeBtn.innerHTML = 'Transcode remote file (single-threaded)';
        await ffmpeg.load({
            workerLoadURL: await toBlobURL(`${baseURLFFMPEG}/814.ffmpeg.js`, 'text/javascript'),
            coreURL: await toBlobURL(`${baseURLCore}/ffmpeg-core.js`, 'text/javascript'),
            wasmURL: await toBlobURL(`${baseURLCore}/ffmpeg-core.wasm`, 'application/wasm'),
        });
    }
    console.log('ffmpeg.load success');
    transcodeBtn.removeAttribute('disabled');
    localFileInput.removeAttribute('disabled');
    remoteFileInput.removeAttribute('disabled');
}

const transcodeRemoteFile = async () => {
    transcodeBtn.setAttribute('disabled', true);
    localFileInput.setAttribute('disabled', true);
    await ffmpeg.writeFile('input.webm', await fetchFile('https://raw.githubusercontent.com/ffmpegwasm/testdata/master/Big_Buck_Bunny_180_10s.webm'));
    await ffmpeg.exec(['-i', 'input.webm', 'output.mp4']);
    const data = await ffmpeg.readFile('output.mp4');
    videoEl.src = URL.createObjectURL(new Blob([data], { type: 'video/mp4' }));
    transcodeBtn.removeAttribute('disabled');
    localFileInput.removeAttribute('disabled');
}

const transcodeLocalFileInput = async () => {
    let files = localFileInput.files;
    let file = files.length ? files[0] : null;
    if (!file) return;
    transcodeBtn.setAttribute('disabled', true);
    localFileInput.setAttribute('disabled', true);
    const data = await transcodeFile(file);
    videoEl.src = URL.createObjectURL(new Blob([data], { type: 'video/mp4' }));
    transcodeBtn.removeAttribute('disabled');
    localFileInput.removeAttribute('disabled');
};

const transcodeFile = async (file) => {
    console.log('file', file);
    const inputDir = '/input';
    const inputFile = `${inputDir}/${file.name}`;
    const outputFile = 'output.mp4';
    console.log('inputFile', inputFile);
    await ffmpeg.createDir(inputDir);
    const workerFSSupported = !!ffmpeg.mount;
    if (workerFSSupported) {
        await ffmpeg.mount('WORKERFS', { files: [file], }, inputDir);
    } else {
        await ffmpeg.writeFile(inputFile, new Uint8Array(await file.arrayBuffer()))
    }
    await ffmpeg.exec(['-i', inputFile, outputFile]);
    const data = await ffmpeg.readFile(outputFile);
    await ffmpeg.deleteFile(outputFile);
    if (workerFSSupported) {
        await ffmpeg.unmount(inputDir);
    } else {
        await ffmpeg.deleteFile(inputFile);
    }
    await ffmpeg.deleteDir(inputDir);
    return data;
};

addEventListener("load", async (event) => {
    localFileInput = document.querySelector('#local-file');
    localFileInput.addEventListener('change', async () => await transcodeLocalFileInput());
    remoteFileInput = document.querySelector('#remote-file');
    loadBtn = document.querySelector('#load-button');
    loadBtn.addEventListener('click', async () => await load());
    loadBtn.removeAttribute('disabled');
    transcodeBtn = document.querySelector('#transcode-button');
    transcodeBtn.addEventListener('click', async () => await transcodeRemoteFile());
    logDiv = document.querySelector('#log-div');
    videoEl = document.querySelector('#video-result');
    logDiv.innerHTML = window.crossOriginIsolated ? 'Multithreading supported' : 'Multithreading not supported';
    console.log('window loaded');
});

```