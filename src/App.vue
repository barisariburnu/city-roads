<template>
  <find-place v-if='!placeFound' @loaded='onGridLoaded'></find-place>
  <div id="app">
    <div v-if='placeFound'>
      <div class='controls'>
        <a href="#" class='print-button' @click.prevent='toggleSettings'>Customize...</a>
        <a href="#" class='try-another' @click.prevent='startOver'>Try another city</a>
      </div>
      <div v-if='showSettings' class='print-window'>
        <h3>Display</h3>
        <div class='row'>
          <div class='col'>Colors</div>
          <div class='col colors c-2'>
            <div v-for='layer in layers' :key='layer.name' class='color-container'>
              <color-picker v-model='layer.color' @change='layer.changeColor'></color-picker>
              <div class='color-label'>{{layer.name}}</div>
            </div>
          </div>
        </div>

        <h3>Export</h3>
        <div class='row'>
          <a href='#'  @click.prevent='toPNGFile' class='col'>As an image (.png)</a> 
          <span class='col c-2'>
            Save the current screen as a raster image.
          </span>
        </div>
        
        <div class='row'>
          <a href='#'  @click.prevent='toSVGFile' class='col'>As a vector (.svg)</a> 
          <span class='col c-2'>
            Save the current screen as a vector image.
          </span>
        </div>
        
        <div class='row'>
          <a href='#'  @click.prevent='toAsphaltSVGFile' class='col'>Asphalt Style</a> 
          <span class='col c-2'>
            Save the current screen as a vector image.
          </span>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import FindPlace from './components/FindPlace.vue';
import LoadingIcon from './components/LoadingIcon.vue';
import EditableLabel from './components/EditableLabel.vue';
import ColorPicker from './components/ColorPicker.vue';
import createScene from './lib/createScene.js';
import GridLayer from './lib/GridLayer.js';
import appState from './lib/appState.js';
import {getPrintableCanvas, getCanvas} from './lib/saveFile.js';
import config from './config.js';
import './lib/canvas2BlobPolyfill.js';
import bus from './lib/bus.js';
import createOverlayManager from './createOverlayManager.js';
import tinycolor from 'tinycolor2';

class ColorLayer {
  constructor(name, color, callback) {
    this.name = name;
    this.changeColor = callback;
    this.color = color;
  }
}

export default {
  name: 'App',
  components: {
    FindPlace,
    LoadingIcon,
    EditableLabel,
    ColorPicker
  },
  data() {
    return {
      placeFound: false,
      name: '',
      showSettings: false,
      settingsOpen: false,
      labelColor: config.getLabelColor().toRgb(),
      backgroundColor: config.getBackgroundColor().toRgb(),
      layers: []
    }
  },
  computed: {
    labelColorRGBA() {
      return toRGBA(this.labelColor);
    }
  },
  created() {
    bus.on('background-color', this.syncBackground);
    bus.on('line-color', this.syncLineColor);
    this.overlayManager = createOverlayManager();
  },
  beforeUnmount() {
    debugger;
    this.overlayManager.dispose();
    this.dispose();
    bus.off('background-color', this.syncBackground);
    bus.off('line-color', this.syncLineColor);
  },
  methods: {
    dispose() {
      if (this.scene) {
        this.scene.dispose();
        window.scene = null;
      }
    },
    toggleSettings() {
      this.showSettings = !this.showSettings;
    },
    onGridLoaded(grid) {
      if (grid.isArea) {
        appState.set('areaId', grid.id);
        appState.unset('osm_id');
        appState.unset('bbox');
      } else if (grid.bboxString) {
        appState.unset('areaId');
        appState.set('osm_id', grid.id);
        appState.set('bbox', grid.bboxString);
      }
      this.placeFound = true;
      this.name = grid.name.split(',')[0];
      let canvas = getCanvas();
      canvas.style.visibility = 'visible';

      this.scene = createScene(canvas);
      this.scene.on('layer-added', this.updateLayers);
      this.scene.on('layer-removed', this.updateLayers);

      window.scene = this.scene;

      let gridLayer = new GridLayer();
      gridLayer.id = 'lines';
      gridLayer.setGrid(grid);
      this.scene.add(gridLayer)
    },

    startOver() {
      appState.unset('areaId');
      appState.unsetPlace();
      appState.unset('q');
      appState.enableCache();

      this.dispose();
      this.placeFound = false;
      this.zazzleLink = null;
      this.showSettings = false;
      this.backgroundColor = config.getBackgroundColor().toRgb();
      this.labelColor = config.getLabelColor().toRgb();

      document.body.style.backgroundColor = config.getBackgroundColor().toRgbString();
      getCanvas().style.visibility = 'hidden';
    },

    toPNGFile(e) {
      scene.saveToPNG(this.name)
    },

    toSVGFile(e) { 
      scene.saveToSVG(this.name)
    },

    toAsphaltSVGFile(e) { 
      this.setBackgroundColor('2e4862')
      scene.saveToSVG(this.name)
    },

    updateLayers() {
      // TODO: This method likely doesn't belong here
      let newLayers = [];
      let lastLayer = 0;
      let renderer = this.scene.getRenderer();
      let root = renderer.getRoot();
      root.children.forEach(layer => {
        if (!layer.color) return;
        let name = layer.id;
        if (!name) {
          lastLayer += 1;
          name = 'lines ' + lastLayer;
        }
        let layerColor = tinycolor.fromRatio(layer.color);
        newLayers.push(new ColorLayer(name, layerColor, newColor => {
          layer.color = toRatioColor(newColor);
          renderer.renderFrame();
          this.scene.fire('color-change', layer);
        }));
      });

      newLayers.push(
        new ColorLayer('background', this.backgroundColor, this.setBackgroundColor),
        new ColorLayer('labels', this.labelColor, newColor => this.labelColor = newColor)
      );

      this.layers = newLayers;

      function toRatioColor(c) {
        return {r: c.r/0xff, g: c.g/0xff, b: c.b/0xff, a: c.a}
      }
    },

    syncLineColor() {
      this.updateLayers();
    },

    syncBackground(newBackground) {
      this.backgroundColor = newBackground.toRgb();
      this.updateLayers()
    },
    setBackgroundColor(c) {
      this.scene.background = c;
      document.body.style.backgroundColor = toRGBA(c);
    }
  }
}

function toRGBA(c) {
    return `rgba(${c.r}, ${c.g}, ${c.b}, ${c.a})`;
}

function recordOpenClick(link) {
  if (typeof ga === 'undefined') return;

  ga('send', 'event', {
      eventCategory: 'Outbound Link',
      eventAction: 'click',
      eventLabel: link
    });
}
</script>

<style lang='stylus'>
@import('./vars.styl');

#app {
  margin: 8px;
  max-height: 100vh;
  position: absolute;
  z-index: 1;
  h3 {
    font-weight: normal;
  }
}

.can-drag {
  border: 1px solid transparent;
}

.drag-overlay {
  position: fixed;
  background: transparent;
  left: 0;
  top: 0;
  right: 0;
  bottom: 0;
}

.overlay-active {
  border: 1px dashed highlight-color;
}
.overlay-active.exclusive {
  border-style: solid;
}

.controls {
  height: 48px;
  background: white;
  display: flex;
  flex-direction: row;
  align-items: stretch;
  width: desktop-controls-width;
  justify-content: space-around;
  box-shadow: 0 2px 4px rgba(0,0,0,0.2), 0 -1px 0px rgba(0,0,0,0.02);

  a {
    text-decoration: none;
    display: flex;
    justify-content: center;
    align-items: center;
    color: highlight-color;
    margin: 0;
    border: 0;
    &:hover {
      color: emphasis-background;
      background: highlight-color;
    }
  }
  a.try-another {
    flex: 1;
  }

  a.print-button {
    flex: 1;
    border-right: 1px solid border-color;
    &:focus {
      border: 1px dashed highlight-color;
    }
  }
}

.col {
    display: flex;
    flex: 1;
    select {
      margin-left: 14px;
    }
  }
.row {
  margin-top: 4px;
  display: flex;
  flex-direction: row;
  min-height: 32px;
}
.colors {
  display: flex;
  flex-direction: row;
  flex-wrap: wrap;

  .color-container {
    display: flex;
    flex-direction: column;
    align-items: center;
    width: 64px;
  }

  .color-label {
    font-size: 12px;
  }
}

a {
  border: 1px solid transparent;
  margin: -1px;
  text-decoration: none;
  color: highlight-color
}
a:focus {
  border: 1px dashed highlight-color;
  outline: none;
}
.print-window {
  max-height: calc(100vh - 48px);
  overflow-y: auto;
  border-top: 1px solid border-color;
  background: white;
  box-shadow: 0 2px 4px rgba(0,0,0,0.2);
  width: desktop-controls-width;
  padding: 8px;
  .row a {
    margin-right: 4px;
  }

  h3 {
    margin: 8px 0;
    text-align: right;
  }
}

.message {
  border-top: 1px solid border-color
  border-bottom: 1px solid border-color
  background: #F5F5F5;
}

.preview-actions {
  display: flex;
  padding: 8px 0;
  margin-left: -8px;
  margin-bottom: 14px;
  margin-top: 1px;
  width: desktop-controls-width;
  flex-direction: column;
  align-items: stretch;
  font-size: 14px;
  align-items: center;
  display: flex;

  .popup-help {
    text-align: center;
  }
}

.city-name {
  position: absolute;
  right: 32px;
  bottom: 54px;
  font-size: 24px;
  color: #434343;
  input {
    font-size: 24px;
  }
}

.license {
  text-align: right;
  position: fixed;
  font-family: labels-font;
  right: 32px;
  bottom: 32px;
  font-size: 12px;
  padding-right: 8px;
  a {
    text-decoration: none;
    display: inline-block;
  }
}

.c-2 {
  flex: 2
}

@media (max-width: small-screen) {
  #app {
    width: 100%;
    margin: 0;

    .preview-actions,.error,
    .controls, .print-window {
      width: 100%;
    }
    .loading-container {
      font-size: 12px;
    }

    .print-window {
      font-size: 14px;
    }

  }
  .city-name  {
    right: 8px;
    bottom: 24px;
  }
  .license  {
    right: 8px;
    bottom: 8px;
  }
}

</style>
