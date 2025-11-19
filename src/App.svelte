<script>
  import { copyFile, readTextFile, writeTextFile, readDir, createDir, removeDir, removeFile, renameFile, exists, BaseDirectory } from '@tauri-apps/api/fs'
  import { convertFileSrc } from '@tauri-apps/api/tauri';
  import { open, save } from '@tauri-apps/api/dialog';
  import { basename } from '@tauri-apps/api/path';
  import { documentDir } from '@tauri-apps/api/path';
  import { v4 as uuidv4 } from 'uuid';
  import axios from 'axios';
  import { Command } from '@tauri-apps/api/shell'
  import 'material-symbols';
  import Tooltip from "sv-tooltip"

  async function startExpressServer() {
    const command = Command.sidecar('binaries/import-export-handler');
    await command.execute()
  }

  startExpressServer()

  let nowPlayingPlaylist;

  function checkInput(event) {
    if (event.data == '`' || event.data == "\\" || event.data === "/" || event.data === '"' || event.data === "'") {
      event.preventDefault();
      dialog("Illegal character", 'negative')
    }
  }

  function checkPlaylistNameInput(event) {
    if (event.data == '`' || event.data == "\\" || event.data === "/" || event.data === '"' || event.data === "'") {
      event.preventDefault();
      dialog("Illegal character", 'negative')
    }

    if (editPlaylistName.length > 63) {
      editPlaylistName = editPlaylistName.slice(0, 63)
      dialog("Reached character limit", 'negative')
    }
    }

  async function checkFiles() {
    let folderExists = await exists('_playlistener', { dir: BaseDirectory.Document })
    let playlistsJsonExists = await exists('_playlistener\\playlists.json', { dir: BaseDirectory.Document })
    if (folderExists == false) {
      await createDir('_playlistener', { dir: BaseDirectory.Document, recursive: true})
      refresh++;
    }

    if (playlistsJsonExists == false) {
      await writeTextFile('_playlistener\\playlists.json', '[]', { dir: BaseDirectory.Document})
      refresh++;
    }


  }

  checkFiles()

  async function getLibrary () {
    let playlistDir = await readTextFile('_playlistener\\playlists.json', {dir: BaseDirectory.Document})
    return JSON.parse(playlistDir)
  }

  async function readPlaylistPath(path) {
    const playlistPath = path + '\\playlist.json'
    const playlistPathArr = [];
    playlistPathArr.push(JSON.parse(await readTextFile(playlistPath)))
    return playlistPathArr[0]
  }

  async function getLibraryInfo(string, num) {
    let playlist = await readTextFile(string + '/playlist-metadata.json')
    let playlistArr = [];
    playlistArr.push(JSON.parse(playlist))
    if (num == 0) {
      console.log(playlistArr[0][0].playlistName)
      return playlistArr[0][0].playlistName
    }
    if (num == 1) {
      console.log(playlistArr[0][0].playlistCover)
      return playlistArr[0][0].playlistCover
    }

  }


  async function getPlaylist(num) {
    let playlistDir = await getLibrary()
    return playlistDir[num].path
  }

  
  async function getPlaylistPath() {
    let playlistPath = await getLibrary()
    let playlistPathArr = [];
    playlistPathArr.push(playlistPath)
    console.log("YEP YEP!", playlistPathArr)
    return playlistPathArr
  }

  async function getCurrentPlaylistPath() {
    return JSON.parse(await readTextFile('_playlistener\\playlists.json', { dir: BaseDirectory.Document}))[currentPlaylist].path
  }

async function returnSpecificPlaylistPath(num) {
  let playlist = JSON.parse(await readTextFile('_playlistener\\playlists.json', { dir: BaseDirectory.Document}))
  return playlist[num].path
}

  async function readPlaylist(num) {
  try {
    let aplaylistdb = await readTextFile(await getPlaylist(num) + '/playlist.json')
    let playlistdb = JSON.parse(aplaylistdb);
    return playlistdb
  } catch (error) {
    console.error("Error reading playlist:", error)
  }
}

async function changePlaylist(num) {
  if (editMode == true) {
    editMode = false
  }
  currentPlaylist = num
  console.log("Changed to " + currentPlaylist)
  console.log(await getLibrary())
  
}

async function readPlaylistMetadata(num) {
  try {
    let aplaylistmdb = await readTextFile(await getPlaylist(num) + '/playlist-metadata.json')
    let playlistmdb = JSON.parse(aplaylistmdb);
    return playlistmdb
  } catch (error) {
    console.error("Error reading playlist metadata:", error)
  }
}

  let currentPlaylist = 0;

  let currentSong;
  let currentQueue = 0;
  let currentPlaylistPath = "";
  let songTime = 0;
  let songSec = 0;
  let songMin = 0;
  let currentSongSec = 0;
  let fixedCurrentSongSec = "00";
  let fixedSongSec = "00";
  let currentSongMin = 0;
  let timer;
  let playpauseVal = "play_arrow"
  let volume = 40;
  let trackArtist = "";
  let trackName = "";
  let trackCover;
  let loopPlaylist = true;
  let Shuffle = false;
  let volumeIcon = "volume_up";
  let shufflePlaylist = [];
  let shuffleNum;
  let shuffleQueue = 0;
  let currentPlaylistPathRaw;

  function toggleShuffle() {
    if (Shuffle == true) {
      Shuffle = false
      shufflePlaylist = [];
      shuffleQueue = 0;
    } else {
      Shuffle = true
    }
  }

  async function updateAudio(track, path) {

    if (path != "0") {
      shuffleQueue = 0;
      shufflePlaylist = [];
      currentPlaylistPath = await readPlaylistPath(path)
      currentPlaylistPathRaw = path
      nowPlayingPlaylist = path
      currentQueue = currentPlaylistPath[track].id
      console.log("paths:", path, currentPlaylistPathRaw)
      console.log(currentQueue)
    } else {
      console.log("No path detected")
    }


    const resolvedQueue = currentPlaylistPath
    console.log(resolvedQueue)
    const audio = document.querySelector('audio');
    const trackText = document.getElementById("trackInfo");
    const trackImgCover = document.getElementById("trackCover");
    if (audio) {
      trackAnim();
      audio.volume = volume / 100;
      audio.src = convertFileSrc(currentPlaylistPathRaw + "/" + resolvedQueue[track].songUrl);
      currentSong = audio.src;
      console.log(resolvedQueue[track].songUrl);
      audio.load();
      audio.play(); 

      if (await exists(currentPlaylistPathRaw + "/" + resolvedQueue[track].songUrl) == false) {
        dialog("This file cannot be played because it is missing or corrupted", "negative")
        
      }

      playpauseVal = "pause"

      function trackAnimReset() {
        trackText.classList.remove("trackChange-a");
        trackText.classList.remove("trackChange-b")
        trackImgCover.classList.remove("coverChange-a");
        trackImgCover.classList.remove("coverChange-b");
      }

      function trackAnim() {
        function anim1() {
          trackText.classList.remove("trackChange-b");
          trackImgCover.classList.remove("coverChange-b");
          trackText.classList.add("trackChange-a");
          trackImgCover.classList.add("coverChange-a");
        }

        function changeTrackInfo() {
          trackArtist = resolvedQueue[track].trackArtist;
          trackName = resolvedQueue[track].trackName;
          trackCover = getSongCover(resolvedQueue[track].cover, currentPlaylistPathRaw);
          console.log(trackCover)
        }

        function anim2() {
          trackImgCover.classList.remove("coverChange-a");
          trackText.classList.remove("trackChange-a");
          changeTrackInfo();
          trackText.classList.add("trackChange-b");
          trackImgCover.classList.add("coverChange-b");
        }

        trackAnimReset();
        setTimeout(anim1, 50);
        setTimeout(anim2, 250);

      }
      
      

      clearInterval(timer);
      timer = setInterval(() => {
        if (audio.paused) return; 
        songTime = Math.floor((audio.currentTime / audio.duration) * 100);
        currentSongMin = Math.floor(audio.currentTime / 60);
        currentSongSec = Math.floor(audio.currentTime - currentSongMin * 60);
        songMin = Math.floor(audio.duration / 60);
        songSec =  Math.floor(audio.duration - songMin * 60);

        if (currentSongSec < 10) {
          fixedCurrentSongSec = "0" + currentSongSec;
        } else {
          fixedCurrentSongSec = currentSongSec.toString();
          fixedSongSec = songSec.toString();
        }

        if (songSec < 10) {
          fixedSongSec = "0" + songSec;
        } else {
          fixedSongSec = songSec.toString();
        }
      
      }, 1);
  }
  }
  async function handleSongEnd() {
    const resolvedQueue = JSON.parse(await readTextFile(currentPlaylistPathRaw + "\\playlist.json"))
    console.log("End:", resolvedQueue)
  if (currentQueue == resolvedQueue.length - 1) {
    console.log("Queue finished!");
    if (loopPlaylist == true) {
      currentQueue = 0;
      updateAudio(currentQueue, 0);
    } else {
      playpauseVal = "play_arrow";
    }
  } else {
    currentQueue++;
    updateAudio(currentQueue, 0);
  }}

function updateSeek(event) {
    const audio = document.querySelector('audio');
    const seekPosition = event.target.value;
    const newTime = (audio.duration * seekPosition) / 100;
    audio.currentTime = newTime;
  }

function PlayPause() {
  const audio = document.querySelector('audio');
  if (playpauseVal == "play_arrow") {
    audio.play();
    playpauseVal = "pause"
  } else {
    audio.pause();
    playpauseVal = "play_arrow"
  }
}

async function queueHandler(num) {
  const resolvedQueue = currentPlaylistPath
  if (resolvedQueue != "") {
  if (Shuffle == false) {
  if (num == 0) {
    currentQueue--;
    if (currentQueue < 0) {
      currentQueue = resolvedQueue.length - 1
    }
    updateAudio(currentQueue, 0)
    console.log("Skipping Backward", currentQueue)
    nowPlayingId.unshift(currentQueue)
    console.log(nowPlayingId)
  }
  if (num == 1) {
    currentQueue++;
    if (currentQueue >= resolvedQueue.length) {
    currentQueue = 0;
  }
    updateAudio(currentQueue, 0)
    console.log("Skipping Forward", currentQueue)
  }
} else {
  if (shufflePlaylist.length == 0) {
    while (shufflePlaylist.length < resolvedQueue.length) {
      shuffleNum = Math.floor(Math.random() * resolvedQueue.length)
      if (shufflePlaylist.includes(shuffleNum) == false) {
        shufflePlaylist.push(shuffleNum)
      }
    }
    console.log("Generated shuffled playlist: " + shufflePlaylist)
  }
  if (num == 0) {
    shuffleQueue--;
    if (shuffleQueue < 0) {
      shuffleQueue = shufflePlaylist.length - 1
    }
    updateAudio(shufflePlaylist[shuffleQueue], 0)
    console.log("Skipping Backward", shufflePlaylist[shuffleQueue])
    currentQueue = shufflePlaylist[shuffleQueue]
    
}
if (num == 1) {
    shuffleQueue++;
    if (shuffleQueue >= shufflePlaylist.length) {
      shuffleQueue = 0
    }
    updateAudio(shufflePlaylist[shuffleQueue], 0)
    console.log("Skipping Forward", shufflePlaylist[shuffleQueue])
    currentQueue = shufflePlaylist[shuffleQueue]


    
}
}
}
}

async function updateVolume(event) {


  const audio = document.querySelector('audio');
  volume = event.target.value;
  console.log("Volume set to " + volume / 100, volume);
  audio.volume = volume / 100;

  if (volume == 0) {
    volumeIcon = "volume_off";
  } else if (volume > 0 && volume <= 50) {
    volumeIcon = "volume_down";
  } else {
    volumeIcon = "volume_up";
  }
}

async function toggleMute() {
  const audio = document.querySelector('audio');
  if (volume > 0) {
    volume = 0;
    audio.volume = 0;
    volumeIcon = "volume_off";
  } else {
    volume = 50;
    audio.volume = 0.5;
    volumeIcon = "volume_up";
  }
}


async function getSongLength(songUrl, path) {
  const audio = document.createElement('audio');
  audio.src = convertFileSrc(path + "/" + songUrl);

  return new Promise((resolve, reject) => {
    audio.onloadedmetadata = function () {
      let minutes = Math.floor(audio.duration / 60)
      let seconds = Math.floor(audio.duration - minutes * 60)
      let fixedSeconds = "00";

      if (seconds < 10) {
        fixedSeconds = "0" + seconds.toString();
      } else {
        fixedSeconds = seconds.toString();
      }

      let length = minutes.toString() + ":" + fixedSeconds.toString()
      resolve(length);
    };
    audio.onerror = function(err) {
      reject(err);
    };
  });
}

function getSongCover(cover, path) {
  let coverImage = convertFileSrc(path + "/" + cover)
  console.log(path + "/" + cover)
  return coverImage
}

let miniDialogText;

let showDialogMessage = false;
let dialogStatus;

function dialog(reason, status) {
  showDialogMessage = true
  miniDialogText = reason
  dialogStatus = status

  setTimeout(timer, 2500)

  function timer() {
    showDialogMessage = false
  }

}

async function getPlaylistName(num) {
  let metadata = await readPlaylistMetadata(num)
  return metadata[0].playlistName
}

async function getPlaylistCover() {
  let metadataPath = await getPlaylistPath()
  let metadata = await readPlaylistMetadata(currentPlaylist)

  return metadataPath[0][currentPlaylist].path + "\\" +  metadata[0].playlistCover
}

async function getPlaylistCoverFile() {
  let metadataPath = await getPlaylistPath()
  let metadata = await readPlaylistMetadata(currentPlaylist)

  return metadata[0].playlistCover
}

async function savePlaylistChanges() {
  let playlistName = (await basename(await getPlaylist(currentPlaylist))).toString()
  console.log(playlistName)
  let metadataPath = '_playlistener\\' + playlistName+ '\\playlist-metadata.json'
  let metadataString = JSON.stringify(await readTextFile(metadataPath, {dir: BaseDirectory.Document}));
  console.log("Metadata String:", metadataString)
  if (editPlaylistName != "") {
    
    if (editPlaylistName != "playlistName") {
    
      dialog("Saved Changes", "positive")

    metadataString = metadataString.replace(`\\"playlistName\\": \\"` + await getPlaylistName(currentPlaylist), `\\"playlistName\\": \\"` + editPlaylistName);
    await writeTextFile(metadataPath, JSON.parse(metadataString), { dir: BaseDirectory.Document})
    console.log("Changed Playlist Name to", editPlaylistName)
    refresh++;

    let imageName = basename(editPlaylistCoverPath)
    console.log("New image name:", (await imageName).toString())

    let playlistPath = await getPlaylist(currentPlaylist)
    let newCoverPath = playlistPath + "\\" + (await imageName + "-cover").toString()
    await copyFile(editPlaylistCoverPath, newCoverPath)
    console.log(editPlaylistCoverPath, newCoverPath)

    metadataString = metadataString.replace(await getPlaylistCoverFile(), (await imageName + "-cover").toString());
    console.log("array v")
    console.log(await getPlaylistCoverFile())
    console.log(JSON.stringify(metadataString))
    await writeTextFile(metadataPath, JSON.parse(metadataString), { dir: BaseDirectory.Document})
    console.log("Changed Playlist Name to", (await imageName + "-cover").toString())
    refresh++;
    
    } else {
      dialog("This playlist name is not allowed", "negative")
    }
  } else {
    console.log("Cannot leave empty.")
    dialog("Cannot leave the Playlist Name field empty", "negative")
  }

}

async function importPlaylistCover() {
  const selected = await open({
  multiple: false,
  filters: [{
    name: 'Image',
    extensions: ['png', 'jpg']
  }]
});
if (selected != null) {
  console.log(selected)
  editPlaylistCover = convertFileSrc(selected);
  editPlaylistCoverPath = selected
}
}

let editSongCover;
let editSongCoverPath = "";
let editSongCoverBasename;

async function importTrackCover() {
  const selected = await open({
  multiple: false,
  filters: [{
    name: 'Image',
    extensions: ['png']
  }]
});
if (selected != null) {
  console.log(selected)
  editSongCover = convertFileSrc(selected);
  editSongCoverPath = selected;
  editSongCoverBasename = await basename(selected);
}
}

let refresh = 0;
let editingPlaylistMetadata = false;
let editPlaylistName = "";
let editPlaylistCover = "";
let editPlaylistCoverPath;

async function editPlaylist() {
  editingPlaylistMetadata = true
  editPlaylistCoverPath = ""
  editPlaylistName = await getPlaylistName(currentPlaylist);
  editPlaylistCover = convertFileSrc(await getPlaylistCover())
}

let editMode = false;
let editingSongInfo = false;
let editSongName;
let editSongId;
let editSongArtist;
let editSongUrl;

function toggleEditMode() {
  if (editMode == true) {
    editMode = false
  } else {
    editMode = true
  }
}

async function editSong(name, image, path, id, artist, url) {
  editingSongInfo = true;
  editSongId = id;
  editSongName = name;
  editSongArtist = artist;
  editSongUrl = url;
  editSongCoverBasename = "";
  editSongCover = getSongCover(image, path)
  console.log(image, path)
}

async function saveSongChanges() {
  let playlistName = (await basename(await getPlaylist(currentPlaylist))).toString()
  let playlistPath = await getPlaylist(currentPlaylist)
  let song = await readPlaylist(currentPlaylist)
  let playlist = JSON.stringify(song)
  let imageUuid = uuidv4()
  console.log(`"trackName":"${song[editSongId].trackName}"`, "to", `"trackName":"${editSongName}"`)

  let songString = JSON.stringify(song[editSongId])
  console.log(songString)

  if (editSongCoverBasename == "") {
    editSongCoverBasename = song[editSongId].cover
  } else {
    console.log("saving song changes:", editSongCoverPath, playlistPath + "\\" + imageUuid)
    await copyFile(editSongCoverPath, playlistPath + "\\" + imageUuid)
  }
  let newSongString = songString.replace(`"trackName":"${song[editSongId].trackName}","trackArtist":"${song[editSongId].trackArtist}","songUrl":"${song[editSongId].songUrl}","cover":"${song[editSongId].cover}"`, `"trackName":"${editSongName}","trackArtist":"${editSongArtist}","songUrl":"${editSongUrl}","cover":"${imageUuid}"`)


  let editedPlaylist = playlist.replace(songString, newSongString)
  console.log(editedPlaylist)

  await writeTextFile('_playlistener\\' + playlistName + '\\playlist.json', editedPlaylist, {dir: BaseDirectory.Document})
  dialog("Saved Changes", "positive")
  refresh++;
  setTimeout(scrollTo, 20);
}


async function moveSong(value, id) {
  let song = await readPlaylist(currentPlaylist)
  let songString = JSON.stringify(song);
  if (value == "up" && (id - 1) != -1) {
    console.log("UP!", song[id], song[id - 1], song[id].id)

    let newstring = songString.replace(`"id":"${song[id].id}","trackName":"${song[id].trackName}","trackArtist":"${song[id].trackArtist}","songUrl":"${song[id].songUrl}","cover":"${song[id].cover}"`, `"id":"${song[id].id}","trackName":"${song[id - 1].trackName}","trackArtist":"${song[id - 1].trackArtist}","songUrl":"${song[id - 1].songUrl}","cover":"${song[id - 1].cover}"`)
    let newstring2 = newstring.replace(`"id":"${song[id - 1].id}","trackName":"${song[id - 1].trackName}","trackArtist":"${song[id - 1].trackArtist}","songUrl":"${song[id - 1].songUrl}","cover":"${song[id - 1].cover}"`, `"id":"${song[id - 1].id}","trackName":"${song[id].trackName}","trackArtist":"${song[id].trackArtist}","songUrl":"${song[id].songUrl}","cover":"${song[id].cover}"`)

    console.log(JSON.parse(newstring2))

    await writeTextFile('_playlistener\\' + (currentPlaylist + 1) + '\\playlist.json', newstring2, {dir: BaseDirectory.Document})
  }

  if (value == "down" && !(id > song.length - 2)) {
    console.log("DOWN!")
    console.log(song[id], song[parseInt(id) + 1])

    let newstring = songString.replace(`"id":"${song[id].id}","trackName":"${song[id].trackName}","trackArtist":"${song[id].trackArtist}","songUrl":"${song[id].songUrl}","cover":"${song[id].cover}"`, `"id":"${song[id].id}","trackName":"${song[parseInt(id) + 1].trackName}","trackArtist":"${song[parseInt(id) + 1].trackArtist}","songUrl":"${song[parseInt(id) + 1].songUrl}","cover":"${song[parseInt(id) + 1].cover}"`)
    let newstring2 = newstring.replace(`"id":"${song[parseInt(id) + 1].id}","trackName":"${song[parseInt(id) + 1].trackName}","trackArtist":"${song[parseInt(id) + 1].trackArtist}","songUrl":"${song[parseInt(id) + 1].songUrl}","cover":"${song[parseInt(id) + 1].cover}"`, `"id":"${song[parseInt(id) + 1].id}","trackName":"${song[id].trackName}","trackArtist":"${song[id].trackArtist}","songUrl":"${song[id].songUrl}","cover":"${song[id].cover}"`)

    await writeTextFile('_playlistener\\' + (currentPlaylist + 1) + '\\playlist.json', newstring2, {dir: BaseDirectory.Document})

  }

  refresh++;
  setTimeout(scrollTo, 50);
}

let playlistYPos;

function scrollY(event) {
  playlistYPos = event.target.scrollTop
}

function scrollTo() {
  document.getElementById('playlist').scrollTop = playlistYPos;
}

let creatingPlaylist = false

async function createPlaylistFolder() {
  let library = await getLibrary()
  let name = uuidv4()
  let newDir = '_playlistener\\' + name
  await createDir(newDir, {dir: BaseDirectory.Document})
  await writeTextFile(newDir + '\\playlist.json', `[]`, {dir: BaseDirectory.Document})
  await writeTextFile(newDir + '\\playlist-metadata.json', JSON.stringify([{"playlistName": 'New Playlist', "playlistCover": 'nocover.png-cover'}], null, 2), {dir: BaseDirectory.Document})
  creatingPlaylist = false;

  let playlistDir = await readTextFile('_playlistener\\playlists.json', {dir: BaseDirectory.Document})
  let playlistArr = [];
  playlistArr.push(JSON.parse(playlistDir))


  let path = await documentDir() + '\\_playlistener\\' + name
  playlistArr[0].push({path: path, name: '' + name + ''})    
  console.log(playlistArr[0])

  await writeTextFile('_playlistener/playlists.json', JSON.stringify(playlistArr[0], null, 2), { dir: BaseDirectory.Document})

  refresh++;
  currentPlaylist = library.length
}

async function importSong() {
  let importedSong;
  const selected = await open({
  multiple: false,
  filters: [{
    name: 'Song (.mp3, .wav, .ogg)',
    extensions: ['mp3', 'wav', 'ogg']
  }]
});
if (selected != null) {
  console.log(selected)
  importedSong = selected

  let playlistArr = JSON.parse(await readTextFile(await getPlaylist(currentPlaylist) + "\\playlist.json"))
  playlistArr.push({id: '' + playlistArr.length + '', trackName: '' + await basename(selected) +'', trackArtist: "N/A", songUrl: '' + await basename(selected) + '', cover: "nocover.png"})
  console.log(playlistArr)

  await writeTextFile(await getPlaylist(currentPlaylist) + "\\playlist.json", JSON.stringify(playlistArr));
  await copyFile(selected, await getPlaylist(currentPlaylist) + "\\" + await basename(selected))
  refresh++;

}
}

async function deleteSong(id) {
  deleteSongId = id
  if (deleteSongDialog == false) {
    deleteSongDialog = true;
  } else {
  let playlist = JSON.parse(await readTextFile(await getPlaylist(currentPlaylist) + "\\playlist.json"))
  let playlistArr = [];

  if (await exists(await getPlaylist(currentPlaylist) + "\\" + playlist[id].songUrl) == true) {
    await removeFile(await getPlaylist(currentPlaylist) + "\\" + playlist[id].songUrl)
  }

  for (let i = 0; i < playlist.length; i++) {
    if (playlist[i] !== playlist[id]) {
        playlistArr.push(playlist[i]);
    }
}

for (let i = 0; i < playlistArr.length; i++) {
  const { id, ...rest } = playlistArr[i];
  playlistArr[i] = rest; 
}

for (let i = 0; i < playlistArr.length; i++) {
  playlistArr[i].id = '' + i + ''; 
}

  await writeTextFile(await getPlaylist(currentPlaylist) + "\\playlist.json", JSON.stringify(playlistArr, null, 2))
  refresh++;
  deleteSongDialog = false
}
}

async function deletePlaylist() {
  if (deletePlaylistDialog == false) {
    deletePlaylistDialog = true
  } else {
  let playlists = JSON.parse(await readTextFile('_playlistener\\playlists.json', { dir: BaseDirectory.Document }))
  let playlistsArr = []
  dialog(`Deleted playlist "${await getPlaylistName(currentPlaylist)}"`, "negative")

  for (let i = 0; i < playlists.length; i++) {
    if (playlists[i] !== playlists[currentPlaylist]) {
      playlistsArr.push(playlists[i])
    }
  }

  console.log(playlistsArr)
  await removeDir(playlists[currentPlaylist].path, {recursive: true})
  await writeTextFile('_playlistener\\playlists.json', JSON.stringify(playlistsArr, null, 2), { dir: BaseDirectory.Document})
  refresh++;
  deletePlaylistDialog = false
  }

}

let addingSongToPlaylist = false;
let selectedSong;

async function getCurrentPlaylistName() {
  return JSON.parse(await readTextFile(await getPlaylist(currentPlaylist) + "\\playlist-metadata.json"))[0].playlistName
}

async function addToPlaylist(song) {
  addingSongToPlaylist = true;
  selectedSong = JSON.stringify(song)
}


async function importSongFromPlaylist(name, path) {
  addingSongToPlaylist = false;
  console.log(selectedSong, "moving to", name, "at", path)
  dialog(`${JSON.parse(selectedSong).trackName} added to ${JSON.parse(await readTextFile(path + '\\playlist-metadata.json'))[0].playlistName}`, "positive")

  let newPlaylist = await readTextFile(path + "\\playlist.json")
  let newPlaylistArr = [];
  newPlaylistArr.push(JSON.parse(newPlaylist))
  newPlaylistArr[0].push(JSON.parse(selectedSong))
  console.log(newPlaylistArr[0])

  let song = JSON.stringify(newPlaylistArr[0][newPlaylistArr[0].length - 1])

  let updatedIdSong = song.replace(`"id":"${JSON.parse(selectedSong).id}"`, `"id":"${newPlaylistArr[0].length - 1}"`)
  newPlaylistArr[0].pop()
  newPlaylistArr[0].push(JSON.parse(updatedIdSong))

  await writeTextFile(path + "\\playlist.json", JSON.stringify(newPlaylistArr[0], null, 2))

  await copyFile(await getPlaylist(currentPlaylist) + "\\" + JSON.parse(updatedIdSong).songUrl, path + "\\" + await basename(JSON.parse(updatedIdSong).songUrl))
  await copyFile(await getPlaylist(currentPlaylist) + "\\" + JSON.parse(updatedIdSong).cover, path + "\\" + await basename(JSON.parse(updatedIdSong).cover))

}

async function exportPlaylist() {
  const filePath = await save({filters: [{ name: '.playlist', extensions: ['playlist'] }]})
  console.log(filePath)
  dialog(`Exporting as ${await basename(filePath)}`, "positive")
  await axios.get('http://127.0.0.1:62024/zip', { params: { value: await getPlaylist(currentPlaylist), toPath: await filePath } }).then(function (response) {
    console.log(response)
    dialog("Export success: " + filePath, "positive")
  })
  .catch(function(error) {
    console.log(error)
    dialog("Failed to export, reason: " + error, "negative")
  })
}

async function importPlaylist() {
  const selected = await open({multiple: false, filters: [{ name: 'Playlist', extensions: ['playlist']}]})
  if (selected != null) {
    let playlistFolderName = await basename(selected, ".playlist");
    console.log(selected, playlistFolderName)
    console.log("Extracted")
    let playlistsJson = await readTextFile('_playlistener\\playlists.json', { dir: BaseDirectory.Document})
    let playlistsArr = []
    playlistsArr.push(JSON.parse(playlistsJson))
    if (JSON.stringify(playlistsArr[0]).includes(playlistFolderName) == true) {
      dialog("Playlist already exists", "negative")
    } else {
      dialog("Importing Playlist", "positive")
    playlistsArr[0].push({path: await documentDir() + '\\_playlistener\\' + playlistFolderName, name: playlistFolderName})
    await writeTextFile(await documentDir() + '\\_playlistener\\playlists.json', JSON.stringify(playlistsArr[0], null, 2))
    console.log(await documentDir() + '\\_playlistener\\' + playlistFolderName)
    
    const response = await axios.get('http://127.0.0.1:62024/unzip', { params: { value: selected, playlistener: await documentDir() + '\\_playlistener\\'}})
    console.log("response:", response.data)
    let playlistCheck;
    playlistCheck = setInterval(async() => {
    console.log("Checking...")
    if (await exists(await documentDir() + '\\_playlistener\\' + playlistFolderName) == false) {
      console.log(await documentDir() + '\\_playlistener\\' + playlistFolderName, "Doesn't exist")
      refresh++;
    } else {
      console.log("Exists")
      dialog("Successfully imported playlist", "positive")
      clearInterval(playlistCheck)
      refresh++;
  }

    }, 100)
    }
  }
}

let deletePlaylistDialog = false;
let deleteSongDialog = false;
let deleteSongId;
let editHover = false;


</script>

{#if showDialogMessage && dialogStatus == "negative"}
<div class="minidialog minidialog-anim negative" id='minidialog-field'>
  {miniDialogText}
</div>
{/if}

{#if showDialogMessage && dialogStatus == "positive"}
<div class="minidialog minidialog-anim positive" id='minidialog-field'>
  {miniDialogText}
</div>
{/if}

{#if deletePlaylistDialog}
<div class="dialog-bg" on:click={() => deletePlaylistDialog = false}></div>
  <div class="dialog-box" style="width:300px;">
    <div style="font-size: 1.5em; text-align: center;">Are you sure you want to delete this playlist?</div>
    <div style="height: 5px">&nbsp;</div>
    <div style="font-size: 1em; text-align: center;">This action is irreversible</div>
    <div style="height: 15px">&nbsp;</div>
    <td width=350 style="text-align: center;"><div class="song-btn danger" on:click={() => deletePlaylist()}>Delete</div></td>
    <td>&nbsp;</td>
    <td width=350 style="text-align: center;"><div class="song-btn" on:click={() => deletePlaylistDialog = false}>Cancel</div></td>
    </div>
{/if}

{#if deleteSongDialog}
<div class="dialog-bg" on:click={() => deleteSongDialog = false}></div>
  <div class="dialog-box" style="width:300px;">
    <div style="font-size: 1.5em; text-align: center;">Are you sure you want to delete this song?</div>
    <div style="height: 5px">&nbsp;</div>
    <div style="font-size: 1em; text-align: center;">This action is irreversible</div>
    <div style="height: 15px">&nbsp;</div>
    <td width=350 style="text-align: center;"><div class="song-btn danger" on:click={() => deleteSong(deleteSongId)}>Delete</div></td>
    <td>&nbsp;</td>
    <td width=350 style="text-align: center;"><div class="song-btn" on:click={() => deleteSongDialog = false}>Cancel</div></td>
    </div>
{/if}

{#if editingPlaylistMetadata}
<div class="dialog-bg" on:click={() => editingPlaylistMetadata = false}></div>
  <div class="dialog-box">
    <span class="material-symbols-rounded" style="position: absolute; transform: translateX(1700%); cursor: pointer;" on:click={() => editingPlaylistMetadata = false}>close</span>
    <table valign="middle">
    <td>
      <div style="background-image: url('placeholder.png'); background-size: cover; width: 180px; height: 180px; border-radius: 10px;">
        <div style="background-image: url('{editPlaylistCover}'); background-size: cover; width: 100%; height: 100%; border-radius: 10px;">
        
        </div>
        
    </div>
    <div style="height: 10px;"></div>

    <div class="song-btn" style="text-align:center;" on:click={() => importPlaylistCover()}>Upload Image</div>

    </td>
    <td>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</td>
    <td width="1000">

      Playlist Name 
      
      {#if editPlaylistName.length < 64}
      <span style="color: #606060;">
      {editPlaylistName.length}/64
    </span>
    {:else}
    <span style="color: #AA0000;">
      {editPlaylistName.length}/64
    </span>
    {/if}
      <div style="height: 10px;"></div>
      {#await getPlaylistName(currentPlaylist)}
        
      {:then playlistName}
      <input type="text" bind:value={editPlaylistName} on:beforeinput={checkPlaylistNameInput} placeholder="Playlist Name">
      {/await}
      <div style="height: 10px;"></div>
      <div class="song-btn" style="text-align:center;" on:click={savePlaylistChanges}>Save Changes</div>
    </td>
  </table>
  </div>
{/if}

{#if editingSongInfo}
<div class="dialog-bg" on:click={() => editingSongInfo = false}></div>
  <div class="dialog-box">
    <span class="material-symbols-rounded" style="position: absolute; transform: translateX(1700%); cursor: pointer;" on:click={() => editingSongInfo = false}>close</span>
    <table valign="middle">
    <td>
      <div style="background-image: url('placeholder.png'); background-size: cover; width: 180px; height: 180px; border-radius: 10px;">
        <div style="background-image: url('{editSongCover}'); background-size: cover; width: 100%; height: 100%; border-radius: 10px;">
        </div>
    </div>
    <div style="height: 10px;"></div>

    <div class="song-btn" style="text-align:center;" on:click={() => importTrackCover()}>Upload Image</div>
    </td>
    <td>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</td>
    <td width="1000">
      Song Name
      <div style="height: 10px;"></div>
      {#await getPlaylistName(currentPlaylist)}
        
      {:then playlistName}
      <input type="text" on:beforeinput={checkInput} bind:value={editSongName} placeholder="Song Name">
      
      {/await}
      <div style="height: 10px;"></div>
      Artist Name
      <div style="height: 10px;"></div>
      <input type="text" on:beforeinput={checkInput} bind:value={editSongArtist} placeholder="Artist Name">

      <div style="height: 10px;"></div>
      <div class="song-btn" style="text-align:center;" on:click={() => saveSongChanges()}>Save Changes</div>
    </td>
  </table>
  </div>
{/if}

{#if addingSongToPlaylist}
<div class="dialog-bg" on:click={() => addingSongToPlaylist = false}></div>

  <div class="dialog-box">
    <span class="material-symbols-rounded" style="position: absolute; transform: translateX(1050%); cursor: pointer;" on:click={() => addingSongToPlaylist = false}>close</span>

    <center>
    <span style="font-size: 1.5em;">Add to Playlist</span>
  </center>
    <div style="height: 15px;"></div>
    <div style="height: 400px; overflow: scroll;">
    {#await getLibrary()}
    {:then playlist}
    {#each playlist as song, index}
    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <!-- svelte-ignore a11y-no-static-element-interactions -->
    {#await getLibraryInfo(song.path, 0)}
    {:then playlistName}
    {#await getCurrentPlaylistName()}
    {:then currentPlaylistName}
    {#await getCurrentPlaylistPath()}
    {:then currentPlaylistPath}
    {#await returnSpecificPlaylistPath(index)}
    {:then playlistPath}
    {#if playlistPath != currentPlaylistPath}
    <div class="song-btn" style="padding: 10px;" on:click={() => importSongFromPlaylist(playlistName, song.path)}>
      {#await getLibraryInfo(song.path, 1)}
      {:then playlistCover}
      <td>
        <div style="background-image: url('placeholder.png'); background-size: cover; width: 40px; height: 40px; border-radius: 5px;">
          {#await getSongCover(playlistCover, song.path)}
          {:then playlistCover}
          <div style="background-image: url('{playlistCover}'); background-size: cover; width: 100%; height: 100%; border-radius: 5px;">
        
          </div>
          {/await}
      </div>
      
    </td>
      {/await}
      <td>&nbsp;&nbsp;</td>
      <td width=200 style="transform: translateY(-10%); font-size: 1.15em;">{playlistName}</td>
    
    </div>
  
    <div style="height: 10px;"></div>
    {/if}
    {/await}
    {/await}
    {/await}
    {/await}
    
  {/each}
  
    {/await}
  </div>
  </div>
  
{/if}

{#key refresh}
<div class="navbar">
  <div style="color: #666666;">
  <td></td>
  <td style="float:left; display:inline-block;"><span class="material-symbols-rounded">library_music</span></td>
  <td>&nbsp;Library</td>
  
  <td style="position: absolute; z-index: 50000000; transform: translate(700%, -75%); cursor: pointer; user-select: none;">
    <Tooltip tip="New" color="#222222" bottom>
    <span class="material-symbols-rounded" on:click={() => createPlaylistFolder()}>add</span>
    </Tooltip>
  </td>

  <td style="position: absolute; z-index: 50000000; transform: translate(600%, -75%); cursor: pointer; user-select: none;">
    <Tooltip tip="Import" color="#222222" bottom>
    <span class="material-symbols-rounded" on:click={() => importPlaylist()}>upload_file</span>
  </Tooltip>
  </td>

</div>

<div style="height:35px"></div>
<div style="overflow-y: scroll; height: 94%; width: 100%; transform: translateY(-3.5%);">
{#key refresh}
  {#await getLibrary()}
  {:then playlist}
  {#each playlist as song, index}
  <!-- svelte-ignore a11y-click-events-have-key-events -->
  <!-- svelte-ignore a11y-no-static-element-interactions -->
  {#await getLibraryInfo(song.path, 0)}
  {:then playlistName}
  <div class="song-btn" on:click={() => changePlaylist(index)}>
    {#await getLibraryInfo(song.path, 1)}
    {:then playlistCover}
    <td>
      <div style="background-image: url('placeholder.png'); background-size: cover; width: 25px; height: 25px; border-radius: 5px;">
        {#await getSongCover(playlistCover, song.path)}
        {:then playlistCover}
        <div style="background-image: url('{playlistCover}'); background-size: cover; width: 100%; height: 100%; border-radius: 5px;">
      
        </div>
        {/await}
    </div>
  </td>
    {/await}
    <td>&nbsp;</td>
    {#if song.path == nowPlayingPlaylist && playpauseVal == "pause"}
    <td style="color: #0072fd;">
      {#if playlistName.length > 32}
      {playlistName.slice(0, 32) + ".."}
      {:else}
      {playlistName}
      {/if}
    </td>
    {:else}
    <td valign="middle">
      {#if playlistName.length > 32}
      {playlistName.slice(0, 32) + ".."}
      {:else}
      {playlistName}
      {/if}
    </td>
    {/if}
  
  </div>
  <div style="height: 10px;"></div>
  {/await}
  
{/each}
  {/await}
  {/key}
</div>
</div>
{#await readPlaylist(currentPlaylist)}
  Loading...
{:then playlist}
{#key refresh}
<div class="playlists" id="playlist" on:scroll={() => scrollY(event)}>
  {#await readPlaylistMetadata(currentPlaylist)}
  Loading...
  {:then metadata}
  {#await getPlaylistPath()}
  {:then playlistPath}
  <div class="playlistInfo-bg" style="background-image: url({getSongCover(metadata[0].playlistCover, playlistPath[0][currentPlaylist].path)})" ></div>
  {/await}
  <div class="playlistInfo-contents" on:click={() => editPlaylist()} on:mouseleave={() => editHover = false} on:mouseenter={() => editHover = true}>
    
    <td>
      {#await getPlaylistPath()}
      {:then playlistPath}
    <!-- svelte-ignore a11y-missing-attribute -->
    <div style="background-image: url('placeholder.png'); cursor:pointer; background-size: cover; width: 150px; height: 150px; border-radius: 10px;)" on:click={() => editPlaylist()}> 
      <div style="background-image: url({getSongCover(metadata[0].playlistCover, playlistPath[0][currentPlaylist].path)}); background-size: cover; width: 150px; height: 150px; border-radius: 10px;)">
    
      </div>
    </div>
    
    {/await}
    
  </td>
  
  <td width=15></td>
  {#if metadata[0].playlistName.length > 24}
  <td valign="bottom">
    <div style="font-size: 2em; font-weight: bold; cursor:pointer;" on:click={() => editPlaylist()}>
      {metadata[0].playlistName.slice(0, 24)}
      <br>
      {metadata[0].playlistName.slice(25, 48)}
      <br>
      {metadata[0].playlistName.slice(49, 64)}

    </div>

    <span>{playlist.length} songs</span>
  </td>
  {:else}
  <td valign="bottom">
    <span style="font-size: 3em; font-weight: bold; cursor:pointer;" on:click={() => editPlaylist()}>
      {metadata[0].playlistName}
  </span>

    <br>
    <span>{playlist.length} songs</span>
  </td>
  {/if}
  {#if editHover}
  <div style="transform: translate(75.5%, -70%);">
<span class="material-symbols-rounded" style="font-size: 0.8em;">edit</span>
<span>&nbsp;Edit Playlist Info</span>

</div>
  {/if}
  </div>
  {/await}
  <div class="songlist">
    <td>
      {#await getPlaylistPath()}
      {:then playlistPath}
      {#if playlistPath[0][currentPlaylist].path == nowPlayingPlaylist && playpauseVal == "pause"}
      <Tooltip tip="Stop Playlist" color="#222222" right>
      <div class="song-btn material-symbols-rounded blue" on:click={() => PlayPause()}>pause</div>
      </Tooltip>
      {:else}
      <Tooltip tip="Play from Beginning" color="#222222" right>
      <div class="song-btn material-symbols-rounded" on:click={() => updateAudio(0, playlistPath[0][currentPlaylist].path)}>play_arrow</div>      
</Tooltip>
      {/if}
      {/await}
    </td>
    <td width=1000>&nbsp;</td>
    <td>
      <Tooltip tip="Add Song" color="#222222" top>
      <div class="song-btn material-symbols-rounded" on:click={() => importSong()}>add</div>
</Tooltip>
    </td>
    <td>&nbsp;</td>
    <td>
      {#if editMode}
      <Tooltip tip="Toggle Edit Mode" color="#222222" top>
      <div class="song-btn material-symbols-rounded active" on:click={() => toggleEditMode()}>edit</div>
    </Tooltip>
{:else}
<Tooltip tip="Toggle Edit Mode" color="#222222" top>

<div class="song-btn material-symbols-rounded" on:click={() => toggleEditMode()}>edit</div>
</Tooltip>

{/if}
    </td>

    <td>&nbsp;</td>
    <td>
      <Tooltip tip="Export Playlist" color="#222222" top>

      <div class="song-btn material-symbols-rounded"  on:click={() => exportPlaylist()}>ios_share</div>
</Tooltip>
    </td>
    <td>&nbsp;</td>
    <td>
      <Tooltip tip="Delete" color="#222222" top>

      <div class="song-btn material-symbols-rounded danger" on:click={() => deletePlaylist()}>delete</div>
      </Tooltip>
    </td>

    <div style="height: 10px;"></div>

  {#each playlist as songs}
  {#if songs.id != undefined}
  <!-- svelte-ignore a11y-click-events-have-key-events -->
  <!-- svelte-ignore a11y-no-static-element-interactions -->
  {#await getPlaylistPath()}
      {:then playlistPath}
{#if editMode}
<div class="editSong-btn">
  <td width="10" on:click={() => updateAudio(songs.id, playlistPath[0][currentPlaylist].path)}></td>
  <td width="30" style="color: #666666;" on:click={() => updateAudio(songs.id, playlistPath[0][currentPlaylist].path)}>{parseInt(songs.id) + 1}</td>
  <td width="50" on:click={() => updateAudio(songs.id, playlistPath[0][currentPlaylist].path)}><div style="border-radius: 5px; width: 35px; height: 35px; background-image: url('placeholder.png'); background-size: cover;">
    <div style="background-image: url({getSongCover(songs.cover, playlistPath[0][currentPlaylist].path)}); background-size: cover; width: 100%; height: 100%; border-radius: 5px;"></div></div></td>
  <td width="250" on:click={() => updateAudio(songs.id, playlistPath[0][currentPlaylist].path)}>
    {songs.trackName}
  </td>
  <td width="20" on:click={() => updateAudio(songs.id, playlistPath[0][currentPlaylist].path)}></td>
  <td width="100" style="color: #666666;" on:click={() => updateAudio(songs.id, playlistPath[0][currentPlaylist].path)}>{songs.trackArtist}</td>

<td>&nbsp;</td>

  <td style="color: #666666;">
    <Tooltip tip="Move Up" color="#222222" top>

    <div class="song-btn material-symbols-rounded" style="font-size: 1em;" on:click={() => moveSong("up", songs.id)}>arrow_upward</div>
    </Tooltip>
  </td>

    <td style="color: #666666;">
      <Tooltip tip="Move Down" color="#222222" top>
      <div class="song-btn material-symbols-rounded" style="font-size: 1em;" on:click={() => moveSong("down", songs.id)}>arrow_downward</div>
      </Tooltip>
    </td>

  <td style="color: #666666;">
    <Tooltip tip="Edit" color="#222222" top>
<div class="song-btn material-symbols-rounded" style="font-size: 1em;" on:click={() => editSong(songs.trackName, songs.cover, playlistPath[0][currentPlaylist].path, songs.id, songs.trackArtist, songs.songUrl)}>edit</div>
</Tooltip>
</td>

<td>

<Tooltip tip="Add to Playlist" color="#222222" top>
  <div class="song-btn material-symbols-rounded" style="font-size: 1em;" on:click={() => addToPlaylist(songs) }>add</div>
  </Tooltip>
</td>
<td>
  <Tooltip tip="Delete" color="#222222" top>
    <div class="song-btn material-symbols-rounded" style="font-size: 1em;" on:click={() => deleteSong(songs.id)} >delete</div>
  </Tooltip>
</td>
</div>
{:else}
      <div class="song-btn" on:click={() => updateAudio(songs.id, playlistPath[0][currentPlaylist].path)}>
    <td width="10"></td>
    <td width="30" style="color: #666666;">
      {#if playlistPath[0][currentPlaylist].path == nowPlayingPlaylist}
      {#if songs.id == currentQueue || songs.id == shufflePlaylist[shuffleQueue]}
      {#if playpauseVal == "pause" }
      <span class="material-symbols-rounded playing" style="color: #0072fd; transform: translate(-30%, 10%)">
        volume_up
      </span>
      {:else}
      {parseInt(songs.id) + 1}
      {/if}
      {:else}
      {parseInt(songs.id) + 1}
      {/if}
      {:else}
      {parseInt(songs.id) + 1}
      {/if}
    </td>

    <td width="50"><div style="border-radius: 5px; width: 35px; height: 35px; background-image: url('placeholder.png'); background-size: cover;"><div style="background-image: url({getSongCover(songs.cover, playlistPath[0][currentPlaylist].path)}); background-size: cover; width: 100%; height: 100%; border-radius: 5px;"></div></div></td>
    <td width="300">
      {songs.trackName}
    </td>
    <td width="20"></td>
    <td width="185" style="color: #666666;">{songs.trackArtist}</td>

    <td style="color: #666666;">
      {#await getSongLength(songs.songUrl, playlistPath[0][currentPlaylist].path)}
        0:00
      {:then duration} 
        {duration}
      {/await}
  </td>
  </div>
  {/if}
  {/await}
  <div style="height: 10px;"></div>
  {/if}
{/each}
</div>
</div>
  {/key}
{/await}
{/key}
<audio on:ended={handleSongEnd} autoplay>
</audio>

<div class="player">
  <div class="track">
  <div class="trackCover" id="trackCover">
    <div style="background-image: url({trackCover}); background-size: cover; width: 100%; height: 100%; border-radius: 10px;">
    </div>
  </div>
  <div class="trackSpacer"></div>
  <div class="trackInfo" id='trackInfo'>
    <div class="trackInfoName">
      {#if trackName.length < 24}
      {trackName}
      {:else}
      {trackName.slice(0, 25) + "..."}
    {/if}
    </div>
    <div class="trackInfoArtist">
      {#if trackArtist.length < 24}
      {trackArtist}
      {:else}
      {trackArtist.slice(0, 25) + "..."}
      {/if}
    </div>
    
  </div>
</div>
<div class="mediaControls">
  <center>
      <!-- svelte-ignore a11y-click-events-have-key-events -->
  <!-- svelte-ignore a11y-no-static-element-interactions -->
  <Tooltip tip="Toggle Shuffle" color="#222222" top>

  {#if Shuffle}
  <span class="mediaButton material-symbols-rounded" style="color:#0072fd;" on:click={() => toggleShuffle()}>shuffle</span>
  {:else}
  <span class="mediaButton material-symbols-rounded" on:click={() => toggleShuffle()}>shuffle</span>
  {/if}
  </Tooltip>
  <!-- svelte-ignore a11y-click-events-have-key-events -->
  <!-- svelte-ignore a11y-no-static-element-interactions -->
  <Tooltip tip="Previous" color="#222222" top>
  <span class="mediaButton material-symbols-rounded" on:click={() => queueHandler(0)}>skip_previous</span>
  </Tooltip>

  <!-- svelte-ignore a11y-click-events-have-key-events -->
  <!-- svelte-ignore a11y-no-static-element-interactions -->
  <Tooltip tip="Play" color="#222222" top>
  <span class="mediaButton material-symbols-rounded" on:click={PlayPause}>{playpauseVal}</span>
    <!-- svelte-ignore a11y-click-events-have-key-events -->
  <!-- svelte-ignore a11y-no-static-element-interactions -->
   </Tooltip>
   <Tooltip tip="Next" color="#222222" top>
  <span class="mediaButton material-symbols-rounded" on:click={() => queueHandler(1)}>skip_next</span>
      <!-- svelte-ignore a11y-click-events-have-key-events -->
  <!-- svelte-ignore a11y-no-static-element-interactions -->
   </Tooltip>
   <Tooltip tip="Toggle Loop" color="#222222" top>
  {#if loopPlaylist}
  <span class="mediaButton material-symbols-rounded" style="color:#0072fd;" on:click={() => loopPlaylist = false}>repeat</span>
  {:else}
  <span class="mediaButton material-symbols-rounded" on:click={() => loopPlaylist = true}>repeat</span>
  {/if}
   </Tooltip>

</center>
<div style="display: block; color: #909090; font-size: 0.6em; font-weight: 2300; transform: translateY(25%);">
  <td width=30 style="transform: translateY(5%);">{currentSongMin}:{fixedCurrentSongSec}</td>
  <td width=250><input type="range" class="mediaSeek" value={songTime} on:input={updateSeek}></td>
  <td width=30 style="transform: translateY(5%); text-align:right;">{songMin}:{fixedSongSec}</td>
  
  
</div>
</div>
<div class="mediaControlsVolume">
    <!-- svelte-ignore a11y-click-events-have-key-events -->
<!-- svelte-ignore a11y-no-static-element-interactions -->
<td style="transform: translateY(10%);">
  <Tooltip tip="Toggle Mute" color="#222222" top>
  <span class="mediaButton material-symbols-rounded" on:click={() => toggleMute()}>{volumeIcon}</span>
  </Tooltip>
</td>
<td width=10></td>
<td style="vertical-align: middle;" width=100>
<input type="range" class="mediaSeekVol" value={volume} on:input={updateVolume}>

</td> 
</div>
</div>