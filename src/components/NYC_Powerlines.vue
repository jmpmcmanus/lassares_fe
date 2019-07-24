<template xmlns:>
  <div id="app" :class="[$options.name]">
    <!-- app map -->
    <vl-map v-if="mapVisible" class="map" ref="map" :load-tiles-while-animating="true" :load-tiles-while-interacting="true"
            @click="onMapClick" data-projection="EPSG:4326" @mounted="onMapMounted">
       <!-- map view aka ol.View -->
      <vl-view ref="mapView" :center.sync="center" :zoom.sync="zoom" :rotation.sync="rotation"></vl-view>

      <!-- click interactions -->
      <vl-interaction-select ref="selectInteraction" :features.sync="selectedFeatures">
        <template slot-scope="select">
          <!-- select styles -->
          <vl-style-box>
            <vl-style-stroke color="#33201e" :width="7"></vl-style-stroke>
            <vl-style-fill :color="[254, 178, 76, 0.7]"></vl-style-fill>
            <vl-style-circle :radius="5">
              <vl-style-stroke color="#9e493e" :width="7"></vl-style-stroke>
              <vl-style-fill :color="[254, 178, 76, 0.7]"></vl-style-fill>
            </vl-style-circle>
          </vl-style-box>
          <vl-style-box :z-index="1">
            <vl-style-stroke color="#d43f45" :width="2"></vl-style-stroke>
            <vl-style-circle :radius="5">
              <vl-style-stroke color="#d43f45" :width="2"></vl-style-stroke>
            </vl-style-circle>
          </vl-style-box>
          <!--// select styles -->

          <!-- selected feature popup -->
          <div v-if="isBox === 'no'">
            <vl-overlay class="feature-popup" v-for="feature in select.features" :key="feature.id" :id="feature.id"
                        :position="pointOnSurface(feature.geometry)" :auto-pan="true" :auto-pan-animation="{ duration: 300 }">
              <template slot-scope="popup">
                <section class="card">
                  <header class="card-header">
                    <p class="card-header-title">
                      Feature ID {{ feature.id }}
                    </p>
                    <a class="card-header-icon" title="Close"
                       @click="selectedFeatures = selectedFeatures.filter(f => f.id !== feature.id)">
                      <b-icon icon="close"></b-icon>
                    </a>
                  </header>
                  <div class="card-content">
                    <div class="content">
                      <div v-if="pid == feature.properties['powerline']">
                        Powerline: {{ powerline }}</br>
                        Voltage: {{ feature.properties['voltage'] }}</br>
                        Service Date: {{ feature.properties['service_date'] }}
                      </div>
                      <div v-else-if="pid == feature.properties['chem_id']">
                        Chemical ID: {{ chem_id }}</br>
                        Concentration: {{ concentrat }}</br>
                        Timestamp: {{ timestamp }}
                        Device ID: {{ feature.properties['device_id'] }}</br>
                        Job ID: {{ feature.properties['job_id'] }}</br>
                        Air Temperature: {{ feature.properties['amb_temp'] }}</br>
                        Air Pressure: {{ feature.properties['air_pressu'] }}</br>
                        Relative Humidity: {{ feature.properties['rel_humid'] }}</br>
                        Precipitation: {{ feature.properties['precip'] }}</br>
                        Wind Speed: {{ feature.properties['wind_speed'] }}</br>
                        Wind Direction: {{ feature.properties['wind_direc'] }}
                      </div>
                      </p>
                    </div>
                  </div>
                </section>
              </template>
            </vl-overlay>
          </div>
          <!--// selected popup -->
        </template>
      </vl-interaction-select>
      <!--// click interactions -->

      <!-- base layers -->
      <vl-layer-tile v-for="layer in baseLayers" :key="layer.name" :id="layer.name" :visible="layer.visible">
        <component :is="'vl-source-' + layer.name" v-bind="layer"></component>
      </vl-layer-tile>
      <!--// base layers -->

      <!-- other layers from config -->
      <component v-for="layer in layers" :is="layer.cmp" v-if="layer.visible" :key="layer.id" v-bind="layer">
        <!-- add vl-source-* -->
        <component ref="layerSource" :is="layer.source.cmp" v-bind="layer.source">
        </component>

        <!-- add style components if provided -->
        <!-- create vl-style-box or vl-style-func -->
        <component v-if="layer.style" v-for="(style, i) in layer.style" :key="i" :is="style.cmp" v-bind="style">
        </component>
        <!--// style -->
      </component>
      <!--// other layers -->

    </vl-map>
    <!--// app map -->

    <!-- menu panel, controls -->
    <div class="menu-panel">
      <b-collapse class="panel box is-paddingless" :open.sync="panelOpen">
        <div slot="trigger" class="panel-heading">
          <div class="columns">
            <div class="column is-11">
              <strong>Menu panel</strong>
            </div>
            <div class="column">
              <b-icon :icon="panelOpen ? 'chevron-up' : 'chevron-down'" size="is-small"></b-icon>
            </div>
          </div>
        </div>
        <p class="panel-tabs">
          <a @click="showMapPanelTab('layers')" :class="mapPanelTabClasses('layers')">Layers</a>
          <a @click="showMapPanelTab('state')" :class="mapPanelTabClasses('state')">State</a>
          <a @click="showMapPanelTab('legend')" :class="mapPanelTabClasses('legend')">Legend</a>
        </p>

        <div class="panel-block" v-for="layer in layers" :key="layer.id" @click="showMapPanelLayer"
             :class="{ 'is-active': layer.visible }"
             v-show="mapPanel.tab === 'layers'">
          <b-switch :key="layer.id" v-model="layer.visible">
            {{ layer.title }}
          </b-switch>
        </div>

        <div class="panel-block" v-show="mapPanel.tab === 'state'">
          <table class="table is-fullwidth">
            <tr>
              <th>Map center</th>
              <td>{{ center }}</td>
            </tr>
            <tr>
              <th>Map zoom</th>
              <td>{{ zoom }}</td>
            </tr>
            <tr>
              <th>Map rotation</th>
              <td>{{ rotation }}</td>
            </tr>
            <tr>
              <th>Device coordinate</th>
              <td>{{ deviceCoordinate }}</td>
            </tr>
            <tr>
              <th>Selected features</th>
              <td>{{ pid }}</td>
            </tr>
          </table>
        </div>

        <div class="panel-block" v-show="mapPanel.tab === 'legend'">
          <table class="table is-fullwidth">
            <tr>
              <b-collapse :open="true">
                <div slot="trigger">
                   <th>NYC Powerlines</th>
                </div>
                <div class="content">
                  <table class="table is-fullwidth">
                    <tr>
                      <!--// style="background-color: hsla(0, 0%, 2%, 1)" &nbsp; -->
                      <td><hr style="border-top: 3px dashed black"/></td>
                      <td>Powerlines</td>
                    </tr>
                    <tr>
                      <td><b>Source</b></td>
                      <td>This data was derived from nothing.
                      </td>
                    </tr>
                  </table>
                </div>
              </b-collapse>
            </tr>
            <tr>
              <b-collapse :open="true">
                <div slot="trigger">
                   <th>PFC 1 Bubbles</th>
                </div>
                <div class="content">
                  <table class="table is-fullwidth">
                    <tr>
                      <!--// style="background-color: hsla(0, 0%, 2%, 1)" &nbsp; -->
                      <td><span class="dot"></span></td>
                      <td>Test Measurments</td>
                    </tr>
                    <tr>
                      <td><b>Source</b></td>
                      <td>This data was derived from nothing.
                      </td>
                    </tr>
                  </table>
                </div>
              </b-collapse>
            </tr>
          </table>
        </div>
      </b-collapse>
    </div>
    <!--// menu panel, controls -->

    <!-- bar panel, controls -->
    <div class="bar-panel">
      <b-collapse class="panel box is-paddingless" :open.sync="barpanelOpen">
        <div slot="trigger" class="panel-heading">
          <div class="columns">
            <div class="column is-11">
              <strong>bar chart panel</strong>
            </div>
            <div class="column">
              <b-icon :icon="barpanelOpen ? 'chevron-up' : 'chevron-down'" size="is-small"></b-icon>
            </div>
          </div>
        </div>

        <div class="panel-block">
          <table class="table is-fullwidth">
            <div v-if="pid == chem_id">
              <div v-if="Object.keys(selectedFeaturesBarClick).length > 0">
                <tr>
                  <th>{{ pid }} Concentration</th>
                </tr>
                <tr>
                  <td>
                    <d3-barchart class="chart" :pdata='selectedFeaturesBarClick' :options='baroptions' />
                  </td>
                </tr>
              </div>
              <div v-else-if="Object.keys(selectedFeaturesBarBox).length > 0">
                <tr>
                  <th>{{ pid }} Concentration</th>
                </tr>
                <tr>
                  <td>
                    <d3-barchart class="chart" :pdata='selectedFeaturesBarBox' :options='baroptions' />
                  </td>
                </tr>
              </div>
            </div>
          </table>
        </div>
      </b-collapse>
    </div>
    <!--// bar panel, controls -->

    <!-- base layers switch -->
    <div class="base-layers-panel">
      <div class="buttons has-addons">
        <button class="button is-light" v-for="layer in baseLayers"
                :key="layer.name" :class="{ 'is-info': layer.visible }"
                @click="showBaseLayer(layer.name)">
          {{ layer.title }}
        </button>
        <button class="button is-light" @click="mapVisible = !mapVisible">
          {{ mapVisible ? 'Hide map' : 'Show map' }}
        </button>
      </div>
    </div>
    <!--// base layers -->
  </div>
</template>

<script>
  import { camelCase } from 'lodash'
  import { findPointOnSurface, writeGeoJsonFeature } from 'vuelayers/lib/ol-ext'
  import ScaleLine from 'ol/control/ScaleLine'
  import FullScreen from 'ol/control/FullScreen'
  import OverviewMap from 'ol/control/OverviewMap'
  import ZoomSlider from 'ol/control/ZoomSlider'
  import { Style, Stroke, Fill, Circle } from 'ol/style'
  import { DEVICE_PIXEL_RATIO } from 'ol/has.js'
  import DragBox from 'ol/interaction/DragBox'
  import { platformModifierKeyOnly } from 'ol/events/condition.js'
  import d3Barchart from '@/mixins/vue-d3-barchart'
  
  export default {
    name: 'nycPowerlines',
    components: {
      d3Barchart,
    },
    data () {
      return {
        center: [-73.851271, 40.725070],
        zoom: 15,
        rotation: 0,
        pid: undefined,
        chem_id: undefined,
        powerline: undefined,
        concentrat: undefined,
        timestamp: undefined,
        selectedFeatures: [],
        selectedFeaturesBarClick: [],
        selectedFeaturesBarBox: [],
        isBox: undefined,
        deviceCoordinate: undefined,
        mapPanel: {
          tab: 'layers',
        },
        panelOpen: true,
        barpanelOpen: true,
        mapVisible: true,
        baroptions: {
          colors: ['blue', 'lightgreen'],
          rules: true,
          axis: true,
          labels: true,
          padding: 0.1,
          line: true,
          points: false,
          value: false,
          gradient: {
            stroke: true,
          },
          curve: {
          },
          getX: (d) => {
            return d.x
          },
          getY: (d) => {
            return d.y
          },
          autoSize: {
            w: 200,
            h: 150,
          },
        },
        baseLayers: [
          {
            name: 'osm',
            title: 'OpenStreetMap',
            visible: true,
          },
          {
            name: 'mapbox',
            title: 'Mapbox Satellite',
            // mapId: 'mapbox.mapbox-streets-v7',
            mapId: 'mapbox.satellite',
            accessToken: 'sk.eyJ1IjoiY29kZWZvcmFtZXJpY2EiLCJhIjoiY2ptd3F1d2Q4MDJ4djNxcjJ6NDltNzhnayJ9.4wsfBXJpT4y9L4tahnag9g',
            visible: false,
          },
        ],
        // layers config
        layers: [
          {
            id: 'nyc_powerlines',
            title: 'NYC Powerlines',
            cmp: 'vl-layer-vector',
            visible: true,
            source: {
              cmp: 'vl-source-vector',
              url: 'http://localhost:8000/api/fdr_18001_0_11_Model/?format=json',
            },
            style: [
              {
                cmp: 'vl-style-func',
                factory: this.getNYC_PowerlinesStyle,
              },
            ],
          },
          {
            id: 'test_measurements',
            title: 'Test Measurements',
            cmp: 'vl-layer-vector',
            visible: true,
            source: {
              cmp: 'vl-source-vector',
              url: 'http://localhost:8000/api/testdata_Model/?format=json',
            },
            style: [
              {
                cmp: 'vl-style-func',
                factory: this.Test_MeasurementsStyle,
              },
            ],
          },
        ],
      }
    },
    methods: {
      camelCase,
      pointOnSurface: findPointOnSurface,
      getNYC_PowerlinesStyle () {
        let canvas = document.createElement('canvas')
        let context = canvas.getContext('2d')
        let pixelRatio = DEVICE_PIXEL_RATIO

        let pattern = (function () {
          canvas.width = 8 * pixelRatio
          canvas.height = 8 * pixelRatio
          // white background
          context.fillStyle = 'rgb(191, 195, 199)'
          context.fillRect(0, 0, canvas.width, canvas.height)
          // outer circle
          context.fillStyle = 'rgb(46, 9, 46, 0.5)'
          context.beginPath()
          context.arc(4 * pixelRatio, 4 * pixelRatio, 3 * pixelRatio, 0, 2 * Math.PI)
          context.fill()
          // inner circle
          context.fillStyle = 'rgb(26, 6, 69)'
          context.beginPath()
          context.arc(4 * pixelRatio, 4 * pixelRatio, 1.5 * pixelRatio, 0, 2 * Math.PI)
          context.fill()
          return context.createPattern(canvas, 'repeat')
        }())
        return feature => {
          return [
            new Style({
              stroke: new Stroke({
                color: pattern,
                width: (this.zoom / 3.7142),
                lineCap: 'round',
                lineJoin: 'bevel',
              }),
            }),
          ]
        }
      },
      Test_MeasurementsStyle () {
        return feature => {
          return [
            new Style({
              image: new Circle({
                radius: (this.zoom / 2.6) + feature.get('concentrat'),
                fill: new Fill({ color: 'rgba(245, 111, 66,0.7)' }),
                stroke: new Stroke({
                  color: 'rgba(245, 111, 66,0.7)',
                  width: 1,
                }),
              }),
            }),
          ]
        }
      },
      /* onUpdatePosition (coordinate) {
        this.deviceCoordinate = coordinate
      }, */
      onMapMounted (map) {
        // now ol.Map instance is ready and we can work with it directly
        this.$refs.map.$map.getControls().extend([
          new ScaleLine(),
          new FullScreen(),
          new OverviewMap({
            collapsed: false,
            collapsible: true,
          }),
          new ZoomSlider(),
        ])

        // a DragBox interaction used to select features by drawing boxes
        const dragBox = new DragBox({
          condition: platformModifierKeyOnly,
        })

        map.$map.addInteraction(dragBox)

        dragBox.on('boxend', () => {
          // features that intersect the box are added to the collection of
          // selected features
          const extent = dragBox.getGeometry().getExtent()
          // only use source that have chem_id
          let source
          var i
          for (i = 0; i < this.$refs.layerSource.length; i++) {
            let key = Object.keys(this.$refs.layerSource[i].$source.featuresRtree_.items_)[0]
            if (this.$refs.layerSource[i].$source.featuresRtree_.items_[key].value.values_.chem_id) {
              source = this.$refs.layerSource[i].$source
            }
          }

          this.selectedFeaturesBarClick = []
          this.isBox = 'yes'

          source.forEachFeatureIntersectingExtent(extent, feature => {
            feature = writeGeoJsonFeature(feature)
            this.selectedFeatures.push(feature)
            this.selectedFeaturesBarBox.push({ x: feature.properties['timestamp'], y: feature.properties['concentrat'] })
            this.chem_id = feature.properties['chem_id']
            this.pid = this.chem_id
          })
        })

        // clear selection when drawing a new box and when clicking on the map
        dragBox.on('boxstart', () => {
          this.selectedFeatures = []
          this.selectedFeaturesBarBox = []
          this.isBox = 'no'
        })
      },
      // base layers
      showBaseLayer (name) {
        let layer = this.baseLayers.find(layer => layer.visible)
        if (layer != null) {
          layer.visible = false
        }

        layer = this.baseLayers.find(layer => layer.name === name)
        if (layer != null) {
          layer.visible = true
        }
      },
      // map panel
      mapPanelTabClasses (tab) {
        return {
          'is-active': this.mapPanel.tab === tab,
        }
      },
      showMapPanelLayer (layer) {
        layer.visible = !layer.visible
      },
      showMapPanelTab (tab) {
        this.mapPanel.tab = tab
      },
      onMapClick (event) {
        let pixel = event.pixel
        let features = this.$refs.map.$map.getFeaturesAtPixel(pixel)

        if (!features) {
          this.selectedFeaturesBarClick = []
          this.selectedFeaturesBarBox = []
          this.selectedFeatures = []
          this.isBox = 'no'
        } else if (features) {
          this.deviceCoordinate = event.coordinate
          let feature = features[0]
          let properties = feature.getProperties()

          if (properties['chem_id']) {
            this.pid = properties['chem_id']
            this.chem_id = this.pid
            this.concentrat = properties['concentrat']
            this.timestamp = properties['timestamp']
            this.selectedFeaturesBarClick.push({ x: this.timestamp, y: this.concentrat })
            this.selectedFeaturesBarBox = []
            this.isBox = 'no'
          } else if (properties['powerline']) {
            this.pid = properties['powerline']
            this.powerline = this.pid
            this.concentrat = undefined
            this.timestamp = undefined
            this.selectedFeaturesBarClick = []
            this.selectedFeaturesBarBox = []
            this.isBox = 'no'
          }
        }
      },
    },
  }
</script>

<style lang="sass">
  @import ~bulma/sass/utilities/_all

  html, body, #app
    width: 100%
    height: 100%
    margin: 0
    padding: 0

  .nycPowerlines
    position: relative

    .map
      height: 100%
      width: 100%

    .menu-panel
      padding: 0

      .panel-heading
        box-shadow: 0 .25em .5em transparentize($dark, 0.8)

      .panel-content
        background: $white
        box-shadow: 0 .25em .5em transparentize($dark, 0.8)

      .panel-block
        &.draw-panel
          .buttons
            .button
              display: block
              flex: 1 1 100%

      +tablet()
        position: absolute
        top: 0
        right: 0
        max-height: 50px
        width: 22em

    .bar-panel
      padding: 0

      .panel-heading
        box-shadow: 0 .25em .5em transparentize($dark, 0.8)

      .panel-content
        background: $white
        box-shadow: 0 .25em .5em transparentize($dark, 0.8)

      .panel-block
        &.draw-panel
          .buttons
            .button
              display: block
              flex: 1 1 100%

      +tablet()
        position: absolute
        top: 0
        left: 0
        max-height: 50px
        width: 32em

    .base-layers-panel
      position: absolute
      left: 50%
      bottom: 20px
      transform: translateX(-50%)

    .feature-popup
      position: absolute
      left: -50px
      bottom: 12px
      width: 20em
      max-width: none

      &:after, &:before
        top: 100%
        border: solid transparent
        content: ' '
        height: 0
        width: 0
        position: absolute
        pointer-events: none
      &:after
        border-top-color: white
        border-width: 10px
        left: 48px
        margin-left: -10px
      &:before
        border-top-color: #cccccc
        border-width: 11px
        left: 48px
        margin-left: -11px

      .card-content
        max-height: 20em
        overflow: auto

      .content
        word-break: break-all

  .dot
    height: 15px;
    width: 15px;
    background-color: #f56f4270;
    border-radius: 50%;
    display: inline-block

  .chart
    display: inline-block
    width: 600px
    height: 200px
    margin-top: 0em

</style>
