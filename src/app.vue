<template>
  <div class="wrapper">
    <div class="header">
      <form action="#" v-on:submit.prevent="onSubmit">
        <div class="d-flex">
          <div class="p-2 flex-shrink-1">
            <button class="btn btn-link ml-2" @click="goUpFolder(source)">↑</button>
          </div>
          <div class="p-2 w-100">
            <input type="text" v-model="source" class="form-control">
          </div>
          <div class="p-2 flex-shrink-1">
            <button class="btn btn-outline-primary ml-2" @click="requestGenerateFileTreeObject(source)">Refresh</button>
          </div>
        </div>
      </form>
    </div>

    <div class="d-flex flex-row align-items-stretch">
      <div class="sidebar flex-column flex-shrink-1">
        <h3>Video player</h3>
        <ul class="nav">
          <template v-for="(file, index) in files">
            <a v-if="file.prefix" @click="requestGenerateFileTreeObject(source + '/' + file.prefix)">{{ file.prefix }}</a>
            <li class="nav-item" :class="{ active: file.path == currentFile.path, prefixed: prefixed }" v-if="file.isValid()">
              <file-component classes="nav-link" :file="file" :index="index" :play="play"></file-component>
            </li>
          </template>
      </ul>
    </div>
    <div class="content flex-column flex-fill">
      <div v-if="currentFile">
        <h3 class="mb-3">{{ currentFile.name }}</h3>
        <div class="embed-responsive embed-responsive-16by9">
          <video controls :src="currentFile.path" @click="toggle"></video>
        </div>
        <hr>
        <form action="#" class="form-inline justify-content-end" v-on:submit.prevent="onSubmit">
          Playback speed:
          <select class="form-control ml-2" v-model="speed">
            <option :value="option" v-for="option in [1, 1.25, 1.5, 1.75, 2, 2.5, 3, 4, 5]">{{ option }}</option>
          </select>
        </form>
      </div>
      <div v-else>
        <h3 class="mb-3">Select a video to start</h3>
      </div>
    </div>
  </div>
</div>
</template>

<script>
// const fs = require('fs');
var Promise = require("bluebird");
var fs = Promise.promisifyAll(require("fs")); //This is most convenient way if it works for you
const settings = require('electron-settings');


export default {
  data () {
    return {
      files: [],
      source: settings.get('source') || '~/Desktop',
      speed: settings.get('speed') || 3,
      currentFile: settings.get('currentFile') || {},
      currentIndex: settings.get('currentIndex') || {},
      prefixed: false
    }
  },

  mounted () {
    var _this = this
    this.requestGenerateFileTreeObject(this.source, false);
    this.setupEventListeners();
  },

  watch: {
    currentFile: function() {
      this.setNextFile();
    },

    speed: function() {
      this.setSpeed(this.speed)
      settings.set("speed", this.speed)
    }
  },

  components: {
    FileComponent: require("./file.vue")
  },

  methods:  {
    onSubmit: function() {},

    setupEventListeners: function() {
      document.ondragover = document.ondrop = (ev) => {
        ev.preventDefault()
      }

      document.body.ondrop = (ev) => {
        this.requestGenerateFileTreeObject(ev.dataTransfer.files[0].path)
        ev.preventDefault()
      }
    },

    setNextFile: function() {
      this.nextFile = this.files[this.currentIndex + 1]
    },

    goUpFolder: function() {
      var source = this.source.substr(0, this.source.lastIndexOf("/"));
      this.requestGenerateFileTreeObject(source)
    },

    play: function(index) {
      this.currentFile = this.files[index]
      settings.set("currentFile", this.currentFile)
      this.currentIndex = index
      settings.set("currentIndex", index)
      this.$nextTick(function() {
        this.setupVideo()
        this.setNextFile()
      });
    },

    toggle: function() {
      var video = document.getElementsByTagName("video")[0]
      if(video.paused) {
        video.play()
      } else {
        video.pause()
      }
    },

    setupVideo: function() {
      var _this = this
      var video = document.getElementsByTagName("video")[0]
      if(video) {
        video.playbackRate = this.speed;
      }
      video.onended = function(e) {
        _this.play(_this.currentIndex + 1)
      }
      video.play();
    },

    setSpeed: function(speed) {
      var video = document.getElementsByTagName("video")[0]
      settings.set("speed", speed)
      if(video) {
        video.playbackRate = speed;
      }
    },

    requestGenerateFileTreeObject: function(directoryString, startAtFirst = true) {
      var _this = this
      this.files = []
      this.prefixed = false
      this.source = directoryString.replace(/\/$/g, "")
      settings.set('source', directoryString);
      this.generateFileTreeObject(directoryString).then(function() {
        _this.setSpeed(_this.speed)
        if(startAtFirst) {
          _this.play(0)
        }
      })
    },

    generateFileTreeObject: function(directoryString, prefix = false) {
      var _this = this;
      return fs.readdirAsync(directoryString)
      .then(arrayOfFileNameStrings => {
        const fileDataPromises = arrayOfFileNameStrings.map(fileNameString => {
          const fullPath = `${directoryString}/${fileNameString}`;
          return fs.statAsync(fullPath)
          .then(fileData => {
            const file = {};
            file.filePath = fullPath
            file.prefix = prefix
            prefix = false
            if(fileData.isFile()) {
              _this.files.push(new File(file))
            } else {
              // recurse
              _this.prefixed = true
              return _this.generateFileTreeObject(file.filePath, fileNameString)
              .then(fileNamesSubArray => {
                file.files = fileNamesSubArray;
              })
              .catch(console.error);
            }
            /*End recursive condition*/
            return file;
          })
        });
        return Promise.all(fileDataPromises);
      });
    }
  }
}

class File {
  constructor(file) {
    this.path = file.filePath
    this.name = getFilenameFromPath(this.path)
    this.extension = this.name.split('.').pop()
    this.prefix = file.prefix
  }

  isValid() {
    var isNotHiddenFile = this.name.charAt(0) !== "."
    var isVideoFile = ['mp4', 'mov'].includes(this.extension)
    if(isNotHiddenFile && isVideoFile) {
      return true;
    }
    return false;
  }
}

function getFilenameFromPath(path) {
  return path.replace(/^.*[\\\/]/, '')
}
</script>
