<template xmlns:>
  <div id="app" :class="[$options.name]">
    <!-- app map -->
    <vl-map v-if="mapVisible" class="map" ref="map" :load-tiles-while-animating="true" :load-tiles-while-interacting="true"
            @click="onMapClick" @postcompose="onMapPostCompose"
            data-projection="EPSG:4326" @mounted="onMapMounted">
      <!-- map view aka ol.View -->
      <vl-view ref="view" :center.sync="center" :zoom.sync="zoom" :rotation.sync="rotation"></vl-view>

      <!-- interactions -->
      <!-- v-if="drawType == null" -->
      <vl-interaction-select :features.sync="selectedFeatures">
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
          <!--// selected popup -->
        </template>
      </vl-interaction-select>
      <!--// interactions -->

      <!-- geolocation -->
      <vl-geoloc @update:position="onUpdatePosition">
        <template slot-scope="geoloc">
          <vl-feature v-if="geoloc.position" id="position-feature">
            <vl-geom-point :coordinates="geoloc.position"></vl-geom-point>
            <vl-style-box>
              <vl-style-icon src="../assets/marker.png" :scale="0.4" :anchor="[0.5, 1]"></vl-style-icon>
            </vl-style-box>
          </vl-feature>
        </template>
      </vl-geoloc>
      <!--// geolocation -->

      <!-- base layers -->
      <vl-layer-tile v-for="layer in baseLayers" :key="layer.name" :id="layer.name" :visible="layer.visible">
        <component :is="'vl-source-' + layer.name" v-bind="layer"></component>
      </vl-layer-tile>
      <!--// base layers -->

      <!-- other layers from config -->
      <component v-for="layer in layers" :is="layer.cmp" v-if="layer.visible" :key="layer.id" v-bind="layer">
        <!-- add vl-source-* -->
        <component :is="layer.source.cmp" v-bind="layer.source">
        </component>

        <!-- add style components if provided -->
        <!-- create vl-style-box or vl-style-func -->
        <component v-if="layer.style" v-for="(style, i) in layer.style" :key="i" :is="style.cmp" v-bind="style">
          <!-- create inner style components: vl-style-circle, vl-style-icon, vl-style-fill, vl-style-stroke & etc -->
          <component v-if="style.styles" v-for="(st, cmp) in style.styles" :key="cmp" :is="cmp" v-bind="st">
            <!-- vl-style-fill, vl-style-stroke if provided -->
            <vl-style-fill v-if="st.fill" v-bind="st.fill"></vl-style-fill>
            <vl-style-stroke v-if="st.stroke" v-bind="st.stroke"></vl-style-stroke>
          </component>
        </component>
        <!--// style -->
      </component>
      <!--// other layers -->

    </vl-map>
    <!--// app map -->

    <!-- map panel, controls -->
    <div class="map-panel">
      <b-collapse class="panel box is-paddingless" :open.sync="panelOpen">
        <div slot="trigger" class="panel-heading">
          <div class="columns">
            <div class="column is-11">
              <strong>Map panel</strong>
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
          <a @click="showMapPanelTab('plot')" :class="mapPanelTabClasses('plot')">Plot</a>
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

        <div class="panel-block" v-show="mapPanel.tab === 'plot'">
          <table class="table is-fullwidth">
            <div v-if="pid == chem_id">
              <tr>
                <th>{{ pid }} Concentration</th>
              </tr>
              <tr>
                <td>
                    <d3-barchart class="chart" :data='vtSelection' :options='baroptions' />
                </td>
              </tr>
            </div>
            <div v-else-if="pid == powerline">
              <tr>
                <th>Cannot Plot Powerline</th>
              </tr>
            </div>
          </table>
        </div>
      </b-collapse>
    </div>
    <!--// map panel, controls -->

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
  import { kebabCase, camelCase } from 'lodash'
  import { createProj, addProj, findPointOnSurface, createStyle } from 'vuelayers/lib/ol-ext'
  import ScaleLine from 'ol/control/ScaleLine'
  import FullScreen from 'ol/control/FullScreen'
  import OverviewMap from 'ol/control/OverviewMap'
  import ZoomSlider from 'ol/control/ZoomSlider'
  import { Style, Stroke, Fill, Circle } from 'ol/style'
  import { DEVICE_PIXEL_RATIO } from 'ol/has.js'
  import d3Barchart from '@/mixins/vue-d3-barchart'
  
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

  // Custom projection for static Image layer
  let x = 1024 * 10000
  let y = 968 * 10000
  let imageExtent = [-x / 2, -y / 2, x / 2, y / 2]
  let customProj = createProj({
    code: 'xkcd-image',
    units: 'pixels',
    extent: imageExtent,
  })
  // add to the list of known projections
  // after that it can be used by code
  addProj(customProj)

  const easeInOut = t => 1 - Math.pow(1 - t, 3)

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
        deviceCoordinate: undefined,
        mapPanel: {
          tab: 'layers',
        },
        panelOpen: true,
        mapVisible: true,
        vtSelection: [],
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
        baroptions: {
          rules: true,
          axis: true,
          labels: true,
          padding: 0.2,
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
        },
      }
    },
    methods: {
      camelCase,
      pointOnSurface: findPointOnSurface,
      geometryTypeToCmpName (type) {
        return 'vl-geom-' + kebabCase(type)
      },
      getNYC_PowerlinesStyle () {
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

      onUpdatePosition (coordinate) {
        this.deviceCoordinate = coordinate
      },
      onMapPostCompose ({ vectorContext, frameState }) {
        if (!this.$refs.marker || !this.$refs.marker.$feature) return

        const feature = this.$refs.marker.$feature
        if (!feature.getGeometry() || !feature.getStyle()) return

        const flashGeom = feature.getGeometry().clone()
        const duration = feature.get('duration')
        const elapsed = frameState.time - feature.get('start')
        const elapsedRatio = elapsed / duration
        const radius = easeInOut(elapsedRatio) * 35 + 5
        const opacity = easeInOut(1 - elapsedRatio)
        const fillOpacity = easeInOut(0.5 - elapsedRatio)

        vectorContext.setStyle(createStyle({
          imageRadius: radius,
          fillColor: [119, 170, 203, fillOpacity],
          strokeColor: [119, 170, 203, opacity],
          strokeWidth: 2 + opacity,
        }))

        vectorContext.drawGeometry(flashGeom)
        vectorContext.setStyle(feature.getStyle()(feature)[0])
        vectorContext.drawGeometry(feature.getGeometry())

        if (elapsed > duration) {
          feature.set('start', Date.now())
        }

        this.$refs.map.render()
      },
      onMapMounted () {
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
          this.vtSelection = []
        } else if (features) {
          this.deviceCoordinate = event.coordinate
          let feature = features[0]
          let properties = feature.getProperties()

          if (properties['chem_id']) {
            this.pid = properties['chem_id']
            this.chem_id = this.pid
            this.concentrat = properties['concentrat']
            this.timestamp = properties['timestamp']
            this.vtSelection.push({ x: this.timestamp, y: this.concentrat })
          } else if (properties['powerline']) {
            this.pid = properties['powerline']
            this.powerline = this.pid
            this.concentrat = undefined
            this.timestamp = undefined
            this.vtSelection = []
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

    .map-panel
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
        max-height: 500px
        width: 22em

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
    width: 300px
    height: 100px
    margin-top: 1em

</style>
