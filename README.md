# ![Playlistener](https://github.com/jasonsxyz/playlistener/blob/main/playlistener.png?raw=true)

`play • lis • ten • er`

- play, playlist, listen, listener. (what a revolutionary name!)

A local music player built with Svelte and Tauri.

### Preview

![Image Preview](https://github.com/jasonsxyz/playlistener/blob/main/preview.png?raw=true)

## Features

- Play Songs (duh)
- Shuffle
- Add / Remove Songs
- Modify Songs (Name, Artist, Cover)
- Modify Playlists (Name, Cover)
- Import / Export Playlists (.playlist)

## Blabber

This project was made around 2023 as part of an assignment task for the now defunct HSC course "Software Design and Development". The task was very vague, and our class was simply told to "create an app", nothing more, nothing less. I decided for this task that I wanted to heavily challenge myself and venture forward in more web development technologies, so of course I chose Svelte and Tauri, two frameworks that had caught my eye due to their praise in the community. I think it would've been more wise of me to learn something more hireable within the industry like React and Electron, but I didn't... hah...

I almost got full marks for this assignment, but I ended up losing one mark because I kind of ran out of time to properly polish one of the diagrams required in the portfolio of this project. :(

Nevertheless, this project was a rewarding experience for me, as it made me dive deeper into web development and made me leave my plain HTML + CSS + JS days behind as I explore more frameworks. I sometimes wonder how I even pulled this off, Parkinson's law does wonders.

I've been meaning to open source this project for a while, as my GitHub profile is quite empty.

## Development

To run Playlistener in development, use the following commands to start the Vite server and Tauri application.

```
npm install
npm run tauri dev
```

### Prerequisites

- Node.js ≥ 18 (Vite + Svelte)
- Rust (Tauri)

### System Requirements (Tauri)

Tauri requires the following installed on your system in order to compile and run the app. [Refer to the Tauri documentation for detailed instruction.](https://v2.tauri.app/start/prerequisites/)

- **Windows:** Microsoft C++ Build Tools + WebView2
- **macOS:** Xcode
- **Linux:** webkit2gtk
