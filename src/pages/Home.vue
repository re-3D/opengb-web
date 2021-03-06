<script>
  // libs and services
  require('../services/printer-service.js')
  require('../services/gcode-service.js')
  import PrintStatusCircle from '../components/print/PrintStatusCircle.vue'
  import TemperatureMenu from '../components/temperature/TemperatureMenu.vue'
  import SearchList from '../components/files/SearchList.vue'
  import SlideSwitch from '../components/slideSwitch/SlideSwitch.vue'

  export default {
    components: {
      PrintStatusCircle,
      TemperatureMenu,
      SearchList,
      SlideSwitch
    },
    data () {
      return {
        printerService: printerws,
        files: {},
        selectedFile: printerws.selectedFile,
        uploadFile:{},
        view: 'file-selection',
        fileInfoWidth: '',
        cancel: false,
        del: false
      }
    },
    ready: function () {
      console.log('childNodes: ', this.$el.childNodes)
    },
    computed: {
      isFileSelection: function () {
        console.log(this.view)
        return this.view === 'file-selection'
      },
      isFileInfo: function () {
        return this.view === 'file-info'
      },
      isPrintInfo: function () {
        return this.view === 'print-info'
      },
      showFileInfo: function () {
        var isPrinting = this.printerService.printer.state != 'READY' && this.printerService.printer.state != 'ERROR'
        var hasFile = this.printerService.selectedFile.length > 0 && this.printerService.selectedFile[0] != null
        return isPrinting || hasFile
      },
      showPrintInfo: function () {
        var isPrinting = this.printerService.printer.state != 'READY' && this.printerService.printer.state != 'ERROR'
        return isPrinting
      },
      statusState: function () {
        var s = this.printerService.printer.state
        if (s == 'EXECUTING') {
          s = 'Heating/Printing'
        }
        return s
      },
      motorStatus: function () {
        var test = this.printerService.printer.steppers.enabled
        console.log(this.printerService.printer.steppers)
        return  test ? 'on' : 'off'
      },
      isNotPrinting: function () {
        var s = this.printerService.printer.state
        return s != 'EXECUTING' && s != 'PAUSED'
      }
    },
    methods: {
      setView: function (v) {
        this.view = v
      },
      homePrinter: function () {
        this.printerService.homePrintHead({x:true,y:true,z:true})
      },
      toggleMotors: function () {
        if(this.printerService.printer.steppers.enabled) {
          this.printerService.disengageMotors()
        } else {
          this.printerService.engageMotors()
        }
      },
      deselectFile: function () {
        printerws.deselectFile()
      },
      printFile: function () {
        if (this.selectedFile[0] && this.selectedFile[0].content) {
          printerws.printFile(this.selectedFile[0].id)
        }
      },
      pausePrint: function () {
        printerws.pausePrint()
      },
      resumePrint: function () {
        printerws.resumePrint()
      },
      cancelPrint: function () {
        console.log(this.cancel)
        if (this.cancel) {
          printerws.cancelPrint()
          this.cancel = false
        } else {
          this.cancel = true
          var self = this
          setTimeout(function () { self.cancel = false }, 3000)
        }
      },
      initFileSelect: function () {
        document.getElementById('uploadFile').click();
      },
      onFileChange: function (e) {
        console.log('file changed',e)
        var files = e.target.files || e.dataTransfer.files
        if (files.length) {
          var file;
          var vm = this

          var fr = new FileReader();
          fr.onload = function(e){
            var contents = e.target.result;
            file = {
              'name': files[0].name,
              'contents': contents,
              'image': null,
              'meta': {}
            }
            var gcm = gcodeService.init(contents)
            file.meta = gcm.meta
            vm.saveFile(file);
            e.target.files = null
          }
          fr.readAsText(files[0]);
        } else {
          return;
        }
      },
      saveFile: function (file) {
        printerws.putFile(file)
      },
      deleteSelectedFile: function () {
        console.log('delete file',this.del)
        if (this.del) {
          printerws.deleteFile(this.printerService.selectedFile[0].id)
          this.del = false
        } else {
          this.del = true
          var self = this
          setTimeout(function () { self.del = false }, 3000)
        }
      },
      calcTime: function (seconds) {
        var hours = parseInt( seconds / 3600 ) % 24;
        var minutes = parseInt( seconds / 60 ) % 60;
        var seconds = seconds % 60;
        hours = hours < 10 ? "0" + hours : hours
        minutes = minutes < 10 ? "0" + minutes : minutes
        seconds = seconds < 10 ? "0" + seconds : seconds
        return hours + ':' + minutes + ':' + seconds
      }
    },
    watch: {
      'selectedFile': {
        handler: function (newVal, oldVal) {
          if (newVal[0] && newVal[0].id) {
            this.setView('file-info')
          } else {
            console.log('selectedFile is null', this.selectedFile)
            this.setView('file-selection')
          }
        }
      }
    }

  }
</script>

<template>
  <section id="home-page">

    <div id="temperature-wrapper">
      <div class="temperature-menus">
        <temperature-menu 
          friendly-name="Heated Bed"
          temp-id="bed"
          v-bind:current="printerService.printer.temperatures.bed.current"
          v-bind:target="printerService.printer.temperatures.bed.target"
          >
        </temperature-menu>
        <temperature-menu 
          friendly-name="Extruder 1"
          temp-id="nozzle1"
          v-bind:current="printerService.printer.temperatures.nozzle1.current"
          v-bind:target="printerService.printer.temperatures.nozzle1.target"
          >
        </temperature-menu>
        <temperature-menu 
          friendly-name="Extruder 2"
          temp-id="nozzle2"
          v-bind:current="printerService.printer.temperatures.nozzle2.current"
          v-bind:target="printerService.printer.temperatures.nozzle2.target"
          >
        </temperature-menu>
      </div>
      <button class="button" style="width:100%;" v-on:click="homePrinter()" v-if="isNotPrinting">Home All</button>
      <slide-switch v-on:click="toggleMotors()" v-bind:is-on="motorStatus == 'on'" label-text="motors"></slide-switch>
    </div>

    <div id="file-selection" v-bind:class="{'is-focus': isFileSelection}" v-if="isNotPrinting" v-on:click="setView('file-selection')">
      <div style="height:75px;">
        <button id="file-upload" class="button" type="button" v-on:click="initFileSelect">
          Upload a File
        </button>
        <input 
          id="uploadFile" 
          class="hide"
          type="file"
          accept=".gcode"
          v-on:change="onFileChange"/>
      </div>

      <search-list></search-list>
    </div>

    <div id="file-info" v-if="showFileInfo" v-bind:class="{'is-focus': isFileInfo}" v-on:click="setView('file-info')">

      <span id="file-image" v-bind:class="{hide: selectedFile[0]===null}">
        
        <h2>{{printerService.selectedFile[0]!=null ? printerService.selectedFile[0].name : ''}}</h2>

        <div id="file-meta-data">
          <div>
            <span>Material: </span>
            <span>{{selectedFile[0]['print_material']}}</span>
          </div>
          <div>
            <span>Quality: </span>
            <span>{{selectedFile[0]['print_quality']}}</span>
          </div>
          <div>
            <span>Time: </span>
            <span>{{calcTime(selectedFile[0]['print_time_sec'])}}</span>
          </div>
          <div>
            <span>Filament Length: </span>
            <span>{{selectedFile[0]['print_material_gm']}} gm</span>
          </div>
        </div>

        <button 
          type="button" 
          v-on:click="printFile()" 
          v-bind:class="{hide: printerService.printer.state != 'READY'}" 
          class="button">
          Print
        </button>

        <button 
          type="button" 
          v-on:click="pausePrint()"  
          v-bind:class="{hide: (printerService.printer.state != 'EXECUTING')}" 
          class="button">
          Pause
        </button>
        
        <button type="button" 
          v-on:click="resumePrint()" 
          v-bind:class="{hide: (printerService.printer.state != 'PAUSED')}" 
          class="button">
          Resume
        </button>
        
        <button 
          type="button" 
          v-on:click="cancelPrint()" 
          v-bind:class="{hide: (printerService.printer.state != 'EXECUTING') }" 
          class="button">
          {{cancel ? 'Confirm' :'Stop' }}
        </button>
 
      </span>

      <button 
        type="button" 
        v-on:click="deleteSelectedFile()" 
        class="button alert"
        v-if="isNotPrinting">
        {{del ? 'Confirm' :'Delete File' }}
      </button>

    </div>

    <div id="print-info" v-if="showPrintInfo" v-bind:class="{'is-focus': isPrintInfo}" v-on:click="setView('print-info')">

      <span id="file-image" v-bind:class="{hide: selectedFile[0]===null}">
        
        <print-status-circle 
          v-bind:state="statusState"
          v-bind:size="250" 
          v-bind:value="printerService.printer.print.currentLine" 
          v-bind:goal="printerService.printer.print.totalLines">
        </print-status-circle>

    </div>

  </section>
</template>

<style>

h2{
  text-align: center;
}

#home-page{
  min-height: 100%;
  display: flex;
  flex-direction: row;
  justify-content: flex-start;
  align-content: center;
  padding: 0 1rem;
} 
#temperature-wrapper{
  flex: 0;
}
.temperature-menus{
  display: flex;
  flex-direction: column;
  padding-top: 75px;
}
#file-upload{
  margin: 0px 20px;
  margin-top:22px;
  float:right;
}
#file-selection{
  flex: 1;
  min-width: 250px;
  transition: all .5s;
}
#file-selection.is-focus{
  flex: 1;
  min-width: 70%;
}
#file-info{
  padding-top:75px;
  text-align: center;
  flex: 1;
  transition: all .5s;
}
#file-info.is-focus{
  flex: 1;
}
#print-info{
  padding-top:75px;
  text-align: center;
  flex: 1;
  transition: all .5s;
}
#print-info.is-focus{
  flex: 1;
}
#file-image{
  margin: 20px auto;
}
#file-image h2{
  font-size: 1.7em;
}
#file-info.is-focus #file-image i{
  font-size: 3em;
  margin-bottom: 0;
  line-height: 0;
  color: black;
  position: relative;
  bottom: 30px;
}
#file-image.hide{
  width: 0;
  padding-top: 0;
}
#file-image i{
  font-size: 0px;
}
#file-meta-data{
  font-size: 1.2em;
  margin: 40px 0;
}
.drop-box {
  background: #F8F8F8;
  border: 5px dashed #DDD;
  width: 60%;
  height: 65px;
  text-align: center;
  padding-top: 25px;
  margin: 0px auto;
}
.drop-box.drag-over{
  background: #eee;
  border-color: #cfc;
} 
#files-page{
  position: relative;
  min-height: 100%;
}
#renderArea {
  position: relative;
  left: 0;
  top: 0;
  width: 100%;
  height: calc(100vh - 100px);
  z-index: 0;
}
#renderArea md-progress-circular {
  position: absolute;
  z-index: -1;
  left: 50%;
  top: 50%;
  margin-left: -50px;
  margin-top: -50px;
}

#left-column{
  float: left;
}

#right-column{
  float: left;
}
.button{
  margin-bottom: 7px!important;
}
.button.rounded{
  border-radius:20px;
}

/*@media screen and (max-width: 40em){
  #home-page{
    padding: 0;
  }
  .temperature-menus{
    padding-top: 0 !important;
    flex-direction: row;
  }
}
*/

</style>

