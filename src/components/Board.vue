<template>
  <div class="squares" :style="{'max-height': heightN + 'px', 'background-color': bgcolor}">
    <div class="toolbar">
      <el-button class="action" @click.stop.prevent="saveRemote()" type="secondary" v-if="sharedPad">
          Save
      </el-button>
      <el-button class="action" @click.stop.prevent="zoomOutSquare()" type="primary" v-if="uiState === 2">
          Back <i class="el-icon-zoom-out"></i>
      </el-button>
      <el-dropdown @command="handleDropdownMenu" trigger="click" v-if="uiState !== 2">
        <el-button type="primary">
          Edit<i class="el-icon-arrow-down el-icon--right"></i>
        </el-button>
        <el-dropdown-menu slot="dropdown">
          <template slot="title">Edit</template>
          <el-dropdown-item command="boards">
            <i class="el-icon-document"></i> Board
          </el-dropdown-item>
          <el-dropdown-item command="background">
            <i class="el-icon-picture"></i> Background
          </el-dropdown-item>
          <el-dropdown-item command="download">
            <i class="el-icon-download"></i> Download
          </el-dropdown-item>
          <el-dropdown-item command="load">
            <i class="el-icon-upload"></i> Load
          </el-dropdown-item>
          <el-dropdown-item command="clear">
            <i class="el-icon-remove"></i> Clear
          </el-dropdown-item>
        </el-dropdown-menu>
      </el-dropdown>
      <BoardManager @publishBoard="publishRemote" @changeBoard="changeBoard($event)" @newBoard="createBoard($event)" @close="boardManagerVisible = false" :visible="boardManagerVisible"></BoardManager>
      <LoadFile @updatedState="updateConnections" :boardId="boardId" @close="loadFileVisible = false" :visible="loadFileVisible"></LoadFile>
      <SquareSettings
          :visible="squareSettingsVisible"
          :square="editSquare"

          @close="squareSettingsVisible = false"
          @squareSettingsClose="squareSettingsVisible = false"
          ></SquareSettings
      <ConnectionSettings @connectionSettingsClose="showConnectionEditor = false; connectionSettingsVisible = false" :visible="connectionSettingsVisible" :connection="editConnection"></ConnectionSettings>
      <el-dialog
        title="Background settings"
        :visible.sync="backgroundSettingsVisible"
        :modal="false"
        width="30%">
        <el-color-picker
           show-alpha
           v-model="bgcolor"
           @active-change="onChangeBgcolor"
           :predefine="predefineColors">
        </el-color-picker>
        <p/>
        <el-form>
          <el-form-item label="Background image URL">
            <el-input v-model="bgurl"></el-input>
          </el-form-item>
        </el-form>
        <span slot="footer" class="dialog-footer">
          <el-button @click="backgroundSettingsVisible = false">Cancel</el-button>
          <el-button type="primary" @click="backgroundSettingsVisible = false">Confirm</el-button>
        </span>
      </el-dialog>
    </div>
    <div
      class="board"
      :style="{'position': boardPosition,
               'height': heightN + 'px',
               'width': widthN + 'px',
               'transform-origin': `${zoomX}px ${zoomY}px`,
               'transform': `scale(${zoom}) translateX(${translateX}) translateY(${translateY})`, 'background-image': `url(${bgurl})`,
               'background-color': bgcolor}"
      @mouseup="onStopDrag($event)"
      @mousemove="onDrag($event)"
      @mousedown="onStartDrag($event)"
      >
      <svg @dblclick.prevent.stop="addSquareOnCursor($event)" preserveAspectRatio="xMidYMid meet" :viewBox="`0 0 ${widthN} ${heightN}`" class="backgroundScreen">
        <line v-if="uiState == 5" :x1="connectionTmp[0].x + connectionTmp[0].width/2" :y1="connectionTmp[0].y + connectionTmp[0].height/2" :x2="dragX/zoom" :y2="dragY/zoom" style="stroke:rgb(140, 182, 164);stroke-width:5"/>
        <line @click="onClickConnection($event, c)" :x1="connectionCoords(c)[0][0]" :y1="connectionCoords(c)[0][1]" :x2="connectionCoords(c)[1][0]" :y2="connectionCoords(c)[1][1]" v-for="(c,index) in allConnections(this.boardId)" :key="c.idx" :stroke-dasharray="(c.dashed)?`${c.dashedA}, ${c.dashedB}`:'5,0'" :style="{'stroke': c.color, 'stroke-width': c.width}"/>
      </svg>
      <div v-if="showConnectionEditor" class="connection-editor" :style="{left: conEditorLeft, top: conEditorTop}">
        <el-button class="action" @click="onEditConnection" type="success" icon="el-icon-edit" circle></el-button>
        <el-button class="action" @click="onRemoveConnection" type="danger" icon="el-icon-delete" circle></el-button>
      </div>
      <Square v-for="(s, index) in allSquares(this.boardId)"
              :boardId="boardId"
              @dblClickSquare="openEditor(s, $event)"
              @touchright="onTouchRight()"
              @activated="onActivated"
              @squaresMoved="createConnections(s)"
              :style="{'z-index': s.zIndex}"
              :isDark="squareIsDark()"
              :zoom="zoom"
              :itext="s.text"
              :icolor="s.color"
              :iidx="s.idx"
              :ix="s.x"
              :iy="s.y"
              :iwidth="s.width"
              :iheight="s.height"
              :izIndex="s.zIndex"
              :itextcolor="s.textColor"
              :itextsize="s.textSize"
              :idataUri="s.dataUri"
              :itype="s.type"
              :iborderSize="s.borderSize"
              :iborderColor="s.borderColor"
              :iimageURL="s.imageURL"
              :iwebsiteURL="s.websiteURL"
              :key="s.idx">
      </Square>
    </div>
    <div class="square-border" v-for="(s, index) in allSquares(this.boardId)" :key="s.idx"
        :style="{'top': `${s.y*zoom - 50}px`, 'left': `${(s.x - 10)*zoom}px`, 'width': `${(s.width + 60)*zoom}px`, 'height': '40px'}">
      <el-row class="actions" v-if="s.showActions">
        <!-- <el-button class="action" @click.stop.prevent="openEditor(s, $event)" type="primary" icon="el-icon-edit" circle></el-button> -->
        <el-button class="action" @click="onConnect(s)" type="success" icon="el-icon-share" circle></el-button>
        <el-button class="action" @click="zoomSquare(s)" type="success" icon="el-icon-zoom-in" circle></el-button>
        <el-button class="action" @click="squareSettings(s)" type="success" icon="el-icon-setting" circle></el-button>
        <el-button class="action right" @click="onRemoveSquare(s.idx)" type="danger" icon="el-icon-delete" circle></el-button>
        <!-- <el-button :style="{'background-color': s.color, 'border-color': 'rgba(0,0,0,0.3)'}"  @click="s.selectColor()" type="success" icon="el-icon-edit" circle></el-button> -->
      </el-row>
    </div>
    <div v-if="uiState === 6" class="quickEditor" @click="closeQuickEditor" :style="{height: heightN + 'px', width: widthN + 'px'}">
      <draggable-resizable v-on:dragging="onDragQuickEditor" :x="quickEditorX" :y="quickEditorY" :w="300" :h="300" :parent="true" :zoom="zoom">
        <markdown-editor @mousedown.stop.prevent='quick' ref="quickEdit" :id="'contentEditor'" v-model="newSquareText" :zIndex="20" :autofocus="true"></markdown-editor>
      </draggable-resizable>
    </div>
    <div @click="closeEditorOnClick($event)" class="cover" :style="{'z-index': (showEditor)?'15':'0'}">
      <div class="square-editor" :style="{'opacity': (showEditor)? (editorOpacity): '0', 'visibility': (showEditor)? 'visible' : 'hidden'}">
        <markdown-editor v-if="editSquare.editing" :id="'contentEditor-' + editSquare.idx" :class="{'is-editing': editSquare.editing}" v-model="editSquare.text" :height="300" :zIndex="20"></markdown-editor>
      </div>
    </div>
  </div>
</template>

<script>

import MarkdownEditor from './MarkdownEditor'
import DraggableResizable from './Draggable'
import Square from './Square'
import { mapGetters, mapMutations  } from 'vuex'
import BoardManager from './BoardManager'
import LoadFile from './LoadFiles'
import SquareSettings from './SquareSettings'
import ConnectionSettings from './ConnectionSettings'
import { sideCoords } from '../utils/geom.js'
import Automerge from 'automerge'
import stateTemplate from '../template'
import localforage from 'localforage'
import axios from 'axios'

const uiStates = { 'DEFAULT': 0,
                   'SELECTED_SQUARE': 1,
                   'ZOOM_ON_SQUARE': 2,
                   'EDITING_SQUARE': 3,
                   'DRAG': 4,
                   'CONNECTING': 5,
                   'QUICK_EDIT': 6
                 }

export default {
  name: 'SquareBoard',
  components: {
    Square,
    MarkdownEditor,
    LoadFile,
    SquareSettings,
    ConnectionSettings,
    DraggableResizable,
    BoardManager },
  props: [
    'startPad',
    'iboardId',
    'sharedPad'
  ],
  data: function () {
    return {
      boardId: (this.iboardId)?this.iboardId:'defaultBoard',
      lastZ: 1,
      uiState: uiStates['DEFAULT'],
      squareOnZoom: {},
      automergeState: Automerge.init(),
      selectedConnection: {},
      conEditorTime: 0,
      showConnectionEditor: false,
      loadFileVisible: false,
      boardManagerVisible: false,
      conEditorLeft: 0,
      conEditorTop: 0,
      origWidth: 0,
      origHeight: 0,
      dragX: 0,
      dragY: 0,
      zoomX: 0,
      zoomY: 0,
      newSquareText: '',
      quickEditorX: 0,
      quickEditorY: 0,
      predefineColors: [
        '#ff4500',
        '#ff8c00',
        '#ffd700',
        '#90ee90',
        '#00ced1',
        '#1e90ff',
        '#c71585',
        'rgba(255, 69, 0, 0.68)',
        'rgb(255, 120, 0)',
        'hsv(51, 100, 98)',
        'hsva(120, 40, 94, 0.5)',
        'hsl(181, 100%, 37%)',
        'hsla(209, 100%, 56%, 0.73)',
        '#c7158577'
      ],
      editSquare: {},
      editConnection: {},
      editorOpacity: 1,
      zoom: 1,
      zoomLevel: 0,
      controlDown: false,
      connectionMode: false,
      connectionTmp: [],
      heightNum: 0,
      lastScroll: 0,
      translateX: 0,
      translateY: 0,
      showEditor: false,
      backgroundSettingsVisible: false,
      squareSettingsVisible: false,
      connectionSettingsVisible: false
    }
  },
  beforeCreate () {
  },
  created () {
    if (this.startPad) {
      this.setState({ boardId: this.boardId, newState: this.startPad })
    }

    if (this.sharedPad) {
      this.setState({ boardId: this.boardId, newState: Object.assign({}, stateTemplate['defaultBoard']) })
      axios.get(`https://api.myjson.com/bins/${this.iboardId}`).then(this.updateStateFromRemote)
    }

    window.addEventListener('keydown', this.onKeyDown)
    window.addEventListener('keypress', this.onKeyPress)
    window.addEventListener('scroll', this.onScroll)
    window.addEventListener('resize', this.onResize)
    window.addEventListener('mousemove', this.onMouseMove)
    window.addEventListener('wheel', this.onMouseWheel)

    this.adjustInitialCanvasSize()
    // Get highest z index
    for (var sq in this.allSquares(this.boardId)) {
      if (sq.zIndex > this.lastZ) {
        this.lastZ = sq.zIndex
      }
    }

    this.$nextTick(() => {
      this.updateConnections()
    })

  },
  computed: {
    bgurl: {
      get () {
        return this.$store.state[this.boardId].bgurl
      },
      set (bgurl) {
        this.$store.commit('setBgurl', { boardId: this.boardId, bgurl: bgurl })
      }
    },
    bgcolor: {
      get () {
        return this.$store.state[this.boardId].bgcolor
      },
      set (color) {
        this.$store.commit('setBgcolor', { boardId: this.boardId, color: color })
      }
    },
    heightN: function () {
      return this.height(this.boardId)
    },
    widthN: function () {
      return this.width(this.boardId)
    },
    boardPosition: function () {
      switch (this.uiState) {
        case uiStates['EDITING_SQUARE']:
          return 'fixed'
        default:
          return 'absolute'
      }
    },
    ...mapGetters([
      'allSquares', 'allConnections', 'height', 'width', 'board', 'boardString', 'state'
    ])
  },
  methods: {
    onEditConnection: function () {
      this.connectionSettingsVisible = true
    },
    squareSettings: function (square) {
      this.editSquare = square
      this.squareSettingsVisible = true
    },
    connectionCoords: function (connection) {
      let coords1 = sideCoords(connection.p1, connection.coords[0])
      let coords2 = sideCoords(connection.p2, connection.coords[1])
      return [coords1, coords2]
    },
    squareIsDark: function () {
      if (this.connectionMode) {
        return true
      } else {
        return false
      }
    },
    adjustInitialCanvasSize: function () {
      this.origWidth = window.innerWidth
      this.origHeight = window.innerHeight

      console.log(this.width(this.boardId), this.boardId, this.origWidth)

      if (this.width(this.boardId) > this.origWidth) {
        this.origWidth = this.width(this.boardId)
      } else {
        console.log('SET WIDTH')
        this.setWidth({ boardId: this.boardId, w: this.origWidth })
      }

      if (this.height(this.boardId) > this.origHeight) {
        this.origHeight = this.height(this.boardId)
      } else {
        this.setHeight({ boardId: this.boardId, h: this.origHeight })
      }
    },
    onMouseMove: function (event) {
      this.dragX = event.pageX
      this.dragY = event.pageY
    },
    onStartDrag: function (event) {
      if (event.target.classList[0] === 'backgroundScreen') {
        for (let sq of this.allSquares(this.boardId)) {
          sq.showActions = false
          this.uiState = uiStates['DRAG']
        }
        switch (this.uiState) {
          case 6:
            this.addSquareQuickEditor()
            break
          default:
            this.uiState = uiStates['DRAG']
            this.dragX = event.pageX
            this.dragY = event.pageY
            break
        }
      }
    },
    onDrag: function (event) {
      if (this.uiState === uiStates['DRAG']) {
        let sensibility = 0.7
        if (window.scrollX >= window.scrollMaxX) {
          this.setWidth({ boardId: this.boardId, w: this.width + 400 })
        }
        if (window.scrollY >= window.scrollMaxY) {
          this.setHeight({ boardId: this.boardId, h: this.height + 400 })
        }
        window.scrollBy((this.dragX - event.pageX)*sensibility, (this.dragY - event.pageY)*sensibility)
      }
    },
    onStopDrag: function (event) {
      if (this.uiState === uiStates['DRAG']) {
        this.uiState = uiStates['DEFAULT']
      }
    },
    onTouchRight: function () {
      // this.setWidth(this.width + 200)
      // window.scrollBy(window.scrollX + 200,0)
    },
    closeEditorOnClick: function (event) {
      if (event.target.className === 'cover') {
        this.closeEditor()
        this.boardDisplay = 'block'
      }
    },
    closeEditor: function () {
      this.showEditor = false
      this.uiState = uiStates['DEFAULT']
    },
    openEditor: function (square, e) {
      if (e.stopPropagation) e.stopPropagation()
      if (e.preventDefault) e.preventDefault()

      this.editSquare = square
      this.editSquare.editing = true
      this.showEditor = true
      this.uiState = uiStates['EDITING_SQUARE']
      this.boardDisplay = 'fixed'
    },
    changeEditorOpacity: function () {
      this.editorOpacity = (this.editorOpacity == 1) ? 0.3 : 1
    },
    offsetY: function () {
      return window.scrollY
    },
    offsetX: function () {
      return window.scrollX
    },
    zoomOutSquare: function () {
      this.zoom = 1
      this.translateX = 0
      this.translateY = 0
      this.uiState = uiStates['DEFAULT']
      this.squareOnZoom.showActions = true
    },
    zoomSquare: function (square) {
      this.uiState = uiStates['ZOOM_ON_SQUARE']
      this.squareOnZoom = square
      let scale = (window.innerWidth - 80)/square.width
      square.showActions = false
      this.translateX = this.offsetX()/scale + -square.x + 40/scale + 'px'
      this.translateY = this.offsetY()/scale -square.y  + 40/scale + 'px'
      this.zoom = scale
    },
    updateConnections: function () {
      let cs = this.allConnections(this.boardId)
      if (cs) {
        let new_cs = []
        for (let i = 0; i < cs.length; ++i) {
           let sq1 = this.allSquares(this.boardId).find(s => s.idx == cs[i].p1.idx)
           let sq2 = this.allSquares(this.boardId).find(s => s.idx == cs[i].p2.idx)

           new_cs.push({p1: sq1,
                        p2: sq2,
                        coords: cs[i].coords,
                        color: cs[i].color,
                        width: cs[i].width,
                        dashed: cs[i].dashed,
                        dashedA: cs[i].dashedA,
                        dashedB: cs[i].dashedB})
        }
        this.setConnections({ boardId: this.boardId, connections: new_cs })
      } else {
        this.setConnections({ boardId: this.boardId, connections: [] })
      }
    },
    closeQuickEditor: function (event) {
      if (event.target.className === 'quickEditor') {
        this.addSquareQuickEditor()
      }
    },
    addSquareQuickEditor: function (event) {
      let newSquare = { x: this.quickEditorX/this.zoom,
                        y: this.quickEditorY/this.zoom,
                        text: this.newSquareText,
                        width: 200,
                        height: -1,
                        type: 'markdown',
                        // TODO: Check if the ID already exist
                        idx: Math.random().toString(36).substring(2),
                        zIndex: this.lastZ + 1
                      }
      this.lastZ = this.lastZ + 1
      console.log('Adding to ', this.boardId)
      this.addSquare({ boardId: this.boardId, square: newSquare })
      // this.$refs.quickEdit.value = ''
      this.newSquareText = ''
      this.uiState = uiStates['DEFAULT']
      // let vuex = localStorage.getItemd('vuex')
      // this.$socket.emit('update', JSON.stringify(vuex))
    },
    addSquareOnCursor: function (event) {
      if (event.stopPropagation) event.stopPropagation()
      if (event.preventDefault) event.preventDefault()

      this.uiState = uiStates['QUICK_EDIT']
      this.quickEditorX = (this.offsetX() + event.clientX)
      if (this.quickEditorX + 300 >= window.innerWidth) {
         this.quickEditorX = this.quickEditorX - 300
      }
      this.quickEditorY = (this.offsetY() + event.clientY)
      // this.$refs.quickEdit.focus()

      // let newSquare = { x: (this.offsetX() + event.clientX)/this.zoom,
      //                   y: (this.offsetY() + event.clientY)/this.zoom,
      //                   text: '',
      //                   width: 200,
      //                   height: 200,
      //                   // TODO: Check if the ID already exist
      //                   idx: Math.random().toString(36).substring(2),
      //                   zIndex: this.lastZ + 1
      //                 }
      // this.lastZ = this.lastZ + 1
      // this.addSquare(newSquare)
      // let vuex = localStorage.getItem('vuex')
      // this.$socket.emit('update', JSON.stringify(vuex))
    },
    handleDropdownMenu: function (event) {
      switch (event) {
        case 'boards':
          this.boardManagerVisible = true
          break

        case 'background':
          this.backgroundSettingsVisible = true
          break

        case 'download':
          this.onDownloadPad()
          break

        case 'load':
          this.loadFileVisible = true
          break

        case 'clear':
          this.setState({ boardId: this.boardId, newState: Object.assign({}, stateTemplate['defaultBoard']) })
          this.adjustInitialCanvasSize()
          this.$nextTick(() => {
              this.updateConnections()
          })
          break
      }
    },
    saveRemote: function () {
      if (this.sharedPad) {
        localforage.getItem('vuex').then(s => {
           let padConfig = s
           let boardConfig = s[this.boardId]
           axios.put(`https://api.myjson.com/bins/${this.iboardId}`, boardConfig)
        })
      }
    },
    publishRemote: function (boardId) {
      localforage.getItem('vuex').then(s => {
         let padConfig = s
         let boardConfig = s[boardId]
         boardConfig.shared = true
         var that = this
         axios.post(`https://api.myjson.com/bins`, boardConfig).then((r) => {
          console.log(r.data)
          var remoteId = r.data.uri.split('/').slice(-1)[0]
          boardConfig.shared = true
          this.setState({ boardId: remoteId, newState: Object.assign({}, boardConfig) })
         })
      })
    },
    onChangeBgcolor: function (color) {
      this.bgcolor = color
      // this.$socket.emit('updateBgcolor', color)
    },
    onConnect: function (square) {
      this.connectionMode = true
      this.uiState = uiStates['CONNECTING']
      this.connectionTmp.push(square)
    },
    changeBoard: function (event) {
      this.boardId = event
      //this.$router.push({path: `/b/${this.boardId}`})
      this.adjustInitialCanvasSize()
      // Get highest z index
      for (var sq in this.allSquares(this.boardId)) {
        if (sq.zIndex > this.lastZ) {
          this.lastZ = sq.zIndex
        }
      }
      this.$nextTick(() => {
        this.updateConnections()
        this.$router.push({path: `/b/${this.boardId}`})
      })
    },
    createBoard: function (event) {
      this.setState({ boardId: event, newState: Object.assign({}, stateTemplate['defaultBoard']) })
      this.boardId = event
      this.adjustInitialCanvasSize()
      // Get highest z index
      for (var sq in this.allSquares(this.boardId)) {
        if (sq.zIndex > this.lastZ) {
          this.lastZ = sq.zIndex
        }
      }
      this.$router.push({path: `/b/${this.boardId}`})
    },
    onMouseWheel: function (event) {
      if (this.uiState == uiStates['DEFAULT'] && event.ctrlKey) {
        event.preventDefault()
        if (event.deltaY < 0) {
          // window.scrollBy(event.clientX/this.zoom, 0)
          let oldZoom = this.zoom
          this.changeZoom(-1)

          let diffX = (event.clientX*this.zoom - event.clientX*oldZoom)
          let diffY = (event.clientY*this.zoom - event.clientY*oldZoom)

          window.scrollBy(diffX, diffY)
        } else {
          // window.scrollBy(-event.clientX/this.zoom, 0)
          let oldZoom = this.zoom
          this.changeZoom(1)
          let diffX = (event.clientX*oldZoom - event.clientX*this.zoom)
          let diffY = (event.clientY*oldZoom - event.clientY*this.zoom)
          window.scrollBy(-diffX, -diffY)
        }
      }
    },
    onKeyDown: function (event) {
      switch (this.uiState) {
        case uiStates['DEFAULT']:
          switch (event.key) {
            default:
              break
          }
          break
        case uiStates['ZOOM_ON_SQUARE']:
          switch (event.key) {
            case 'Escape':
              this.zoomOutSquare()
              break
          }
          break
        case uiStates['EDITING_SQUARE']:
          switch (event.key) {
            case 'Escape':
              this.closeEditor()
              break
          }
      }
    },
    onKeyPress: function (event) {
      switch (event.key) {
        case 'z':
          switch (this.uiState) {
            case 0:
              this.changeZoom(+1)
              break
            default:
              break
          }
          break;
        case 'Z':
          switch (this.uiState) {
            case 0:
              this.changeZoom(-1)
              break
            default:
              break
          }
          break
        case 'c':
          switch (this.uiState) {
            case 0:
              this.adjustCanvasSize()
              break
          }
          break
        case 'Enter':
          switch (this.uiState) {
            case uiStates['QUICK_EDIT']:
              this.addSquareQuickEditor()
              break
          }
          break
        default:
          console.log(event.key)
          break
      }
    },
    onResize: function (event) {
      if (this.height(this.boardId) < window.innerHeight) {
        this.setHeight({ boardId: this.boardId, h: window.innerHeight })
      }
      if (this.width(this.boardId) < window.innerWidth) {
        this.setWidth({ boardId: this.boardId, w: window.innerWidth })
      }
    },
    adjustCanvasSize: function () {
      var maxHeight = 0, maxWidth = 0
      for (var sq of this.allSquares(this.boardId)) {
        if (sq.x + sq.width > maxWidth) {
           maxWidth = sq.x + sq.width
        }
        if (sq.y + sq.height > maxHeight) {
           maxHeight = sq.y + sq.height
        }
        if (sq.x + sq.width > this.origWidth) {
          this.origWidth = (sq.x + sq.width + 20)
        }
        if (sq.y + sq.height > this.origHeight) {
          this.origHeight = (sq.y + sq.height + 20)
        }
      }
      if (maxWidth + 20 >= window.innerWidth) {
        this.setWidth({ boardId: this.boardId, w: maxWidth + 20 })
      }

      if (maxHeight + 20 >= window.innerHeight) {
        this.setHeight({ boardId: this.boardId, h: maxHeight + 20 })
      }
    },
    minimumHeight: function () {
       
    },
    changeZoom: function (inc) {
      if (inc >= 1) {
        if (this.zoom === 0.2) {
          return
        }
        this.zoomLevel += 1
        let newZoom = 1  - this.zoomLevel * 0.2
        this.adjustCanvasSize()
        this.zoom = (newZoom >= 0.2) ? newZoom: 0.2
        this.setWidth({ boardId: this.boardId, w: this.origWidth / this.zoom })
        this.setHeight({ boardId: this.boardId, h: this.origHeight / this.zoom })
      } else {
        this.zoomLevel -= 1
        this.adjustCanvasSize()
        if (this.zoomLevel >= 0) {
          this.setWidth({ boardId: this.boardId, w: this.origWidth / (1 - this.zoomLevel*0.2) })
          this.setHeight({ boardId: this.boardId, h: this.origHeight / (1 - this.zoomLevel*0.2) })
        } else {
          this.setWidth({ boardId: this.boardId, w: this.origWidth * (1 - this.zoomLevel*0.2) })
          this.setHeight({ boardId: this.boardId, h: this.origHeight * (1 - this.zoomLevel*0.2) })
        }
        this.zoom = 1 - this.zoomLevel * 0.2
      }
    },
    onActivated: function (event) {
      event.zIndex = this.lastZ
      if (this.connectionMode) {
        let sq1 = this.allSquares(this.boardId).find(s => s.idx == this.connectionTmp[0].idx)
        let sq2 = this.allSquares(this.boardId).find(s => s.idx == event.idx)

        this.addConnection({ boardId: this.boardId, connection: {p1: sq1, p2: sq2} })
        this.connectionMode = false
        this.connectionTmp = []
        this.uiState = uiStates['DEFAULT']
      }
    },
    updateStateFromRemote: function (response) {
      this.setState({ boardId: this.boardId, newState: response.data })
      this.$nextTick(() => {
        this.updateConnections()
      })
    },
    createConnections: function (square) {
      this.updateSquareConnections({ boardId: this.boardId, square: square} )
    },
    onClickLine: function (connection) {
    },
    onRemoveConnection: function () {
      this.showConnectionEditor = false
      this.removeConnection({ boardId: this.boardId, connection: this.selectedConnection })
    },
    onRemoveSquare: function (square_id) {
      this.lastZ -= 1
      this.removeSquare({ boardId: this.boardId, idx: square_id })
    },
    onMouseoverConnection: function (event, connection) {
      this.conEditorTime = new Date().getMilliseconds()
      this.showConnectionEditor = true
      window.setInterval(this.checkTimeoutConEditor, 200)
      this.conEditorLeft = this.offsetX() + event.clientX + 'px'
      this.conEditorTop = this.offsetY() + event.clientY + 'px'
    },
    onClickConnection: function (event, connection) {
      this.editConnection = connection
      this.showConnectionEditor = true
      this.selectedConnection = connection
      this.conEditorLeft = (this.offsetX() + event.clientX)/this.zoom + 'px'
      this.conEditorTop = (this.offsetY() + event.clientY)/this.zoom + 'px'
    },
    onMouseoutConnection: function (event, connection) {
      let now = new Date().getMilliseconds()
      if (now - this.conEditorTime > 1000) {
        this.showConnectionEditor = false
      }
      // let middleX = connection.p1.x + (connection.p1.x - connection.p2.x)/2.0
      // this.conEditorLeft = event.clientX + 'px'
      // this.conEditorTop = this.offsetY() + event.clientY + 'px'
    },
    onDragQuickEditor: function (x, y) {
      this.quickEditorY = y
      this.quickEditorX = x
    },
    checkTimeoutConEditor: function () {
      let now = new Date().getMilliseconds()
      if (now - this.conEditorTime > 300) {
        this.showConnectionEditor = false
        window.clearInterval()
      }
    },
    onScroll: function (event) {
      // if (event.pageY >= this.lastScroll && (window.document.body.scrollHeight === window.scrollY + window.innerHeight)) {
      //   this.changeHeight(25)
      // }

      this.lastScroll = event.pageY
    },
    onDownloadPad: function () {
      let element = document.createElement('a')
      localforage.getItem('vuex').then(s => {
        console.log(s)
        let padConfig = s
        element.setAttribute('href', 'data:text/plaincharset=utf-8,' + encodeURIComponent(JSON.stringify(padConfig[this.boardId])))
        element.setAttribute('download', 'pad-config.json')
        element.style.display = 'none'
        document.body.appendChild(element)
        element.click()
        document.body.removeChild(element)

      })
    },
    ...mapMutations([
      'setBoard', 'addSquare', 'setSquares', 'removeSquare', 'saveSquares', 'addConnection', 'setConnections', 'setHeight', 'changeHeight', 'setWidth', 'removeConnection', 'setState', 'setBgcolor', 'updateSquareConnections'
    ])
  },
  watch: {
    uiState: function (newState, oldState) {
      switch (newState) {
        default:
          switch (oldState) {
            case uiStates['QUICK_EDIT']:
              break
            default:
              break
        }
      }
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="scss">
.squares {

  .board {
    background-color: #cfebff;
    width: 100%;
    top: 0;
    bottom: 0;
    height: 100%;
    left: 0;
    z-index: 10;
    position: absolute;
    width: 100%;
    vertical-align: top;
    // transition: transform 0.5s;
  }
}

.cover {
  display: flex;
  position: absolute;
  z-index: 15;
  top:0;
  left:0;
  bottom:0;
  right: 0;
  width: 100%;
  padding-top: 20px;
  justify-content: center;
}

.square-editor {
  visibility: hidden;
  position: fixed;
  height: 100%;
  opacity: 0;
  transition: z-index 0.3s, visibility 0.3s, opacity 0.3s linear;
  min-height: 50%;
  width: 90%;
  max-height: 90%;
  padding: 20px;
  background-color: white;
}
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}

.square-border {
  position: absolute;
  min-width: 200px;
  z-index: 15;
  display: flex;
  align-items: left;
}

.connection {
   border: 1px solid black;
   position: absolute;
}

.connection-editor {
  position: absolute;
}

.toolbar {
  position: fixed;
  z-index: 100;
  margin: 20px;
}

.el-dialog__wrapper {
  z-index: 10;
}

.el-dropdown-menu__item {
  display: block;
}

.backgroundScreen {
   position: absolute;
   width: 100%;
   height: 100%;
   left:0;
   top:0;
}

.sidebar-menu {
  z-index: 25;

  &.el-menu--collapse {
  }
}

.actions {
  width: 100%;

  .action {
    &.right {
      float: right;
    }
  }
}

.quickEditor {
   position: absolute;
   top: 0;
   left: 0;
   bottom: 0;
   width: 100%;
   height: 100%;
   background: white;
   z-index: 50;
   background: none;
   min-width: 500px;
   input {
      background: white;
      border: none;
      font-size: 36px;
      color: black;
   }
}

</style>
