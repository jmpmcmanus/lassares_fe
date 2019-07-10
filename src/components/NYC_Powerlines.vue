<template xmlns:>
  <div id="app" :class="[$options.name]">
    <!-- app map -->
    <vl-map v-if="mapVisible" class="map" ref="map" :load-tiles-while-animating="true" :load-tiles-while-interacting="true"
            @click="onMapClick" @postcompose="onMapPostCompose"
            data-projection="EPSG:4326" @mounted="onMapMounted">
      <!-- map view aka ol.View -->
      <vl-view ref="view" :center.sync="center" :zoom.sync="zoom" :rotation.sync="rotation"></vl-view>

      <vl-overlay class="feature-popup" v-if="deviceCoordinate" :position="deviceCoordinate" style="background: white; padding: 10px">
        <template slot-scope="popup">
          <section class="card">
            <header class="card-header">
              <a class="card-header-icon" title="Close"
                 @click="onMapDoneClick(), deviceCoordinate = undefined">
                 <b-icon icon="close"></b-icon>
              </a>
                <div v-if="powerline !== undefined">
                  </br>
                  <b>Feeder: {{ title }}</b>
                </div>
                <div v-if="chem_id !== undefined">
                  </br>
                  <b>Measurement:</b> {{ chem_id }}
                </div>
            </header>
            <div class="card-content">
              <div class="content">
                <div v-if="powerline !== undefined">
                  Powerline: {{ powerline }}</br>
                  Voltage: {{ voltage }}</br>
                  Service Date: {{service_date}}
                </div>
                <div v-if="concentrat !== undefined">
                  Concentration: {{ Concentrat }}</br>
                  Timestamp: {{ timestamp }}
                  Device ID: {{ device_id }}</br>
                  Job ID: {{ job_id }}</br>
                  Air Temperature: {{ amb_temp }}</br>
                  Air Pressure: {{ air_pressu }}</br>
                  Relative Humidity: {{ rel_humid }}</br>
                  Precipitation: {{ precip }}</br>
                  Wind Speed: {{ wind_speed }}</br>
                  Wind Direction: {{ wind_direc }}
                </div>
              </div>
            </div>
          </section>
        </template>        
      </vl-overlay>

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
          <a @click="showMapPanelTab('legend')" :class="mapPanelTabClasses('legen')">Legend</a>
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
              <td>{{ powerline }}</td>
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
    data () {
      return {
        center: [-73.851271, 40.725070],
        zoom: 13,
        rotation: 0,
        selectedFeatures: [],
        deviceCoordinate: undefined,
        title: undefined,
        powerline: undefined,
        voltage: undefined,
        service_data: undefined,
        chem_id: undefined,
        concentrat: undefined,
        timestamp: undefined,
        device_id: undefined,
        job_id: undefined,
        amb_temp: undefined,
        air_pressu: undefined,
        rel_humid: undefined,
        precip: undefined,
        wind_speed: undefined,
        wind_direc: undefined,
        mapPanel: {
          tab: 'layers',
        },
        panelOpen: true,
        mapVisible: true,
        vtIdProp: 'id',
        vtSelection: {},
        vtSelectMode: 'single',
        drawType: undefined,
        drawnFeatures: [],
        // base layers
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
      geometryTypeToCmpName (type) {
        return 'vl-geom-' + kebabCase(type)
      },
      getNYC_PowerlinesStyle () {
        return feature => {
          let selected = !!this.vtSelection[feature.get(this.vtIdProp)]

          return [
            new Style({
              stroke: new Stroke({
                color: selected ? 'rgba(0,0,0,1)' : feature.get('color'),
                width: selected ? 3 : 2.5,
                lineDash: [5, 5, 5],
              }),
            }),
          ]
        }
      },
      Test_MeasurementsStyle () {
        return feature => {
          // let selected = !!this.vtSelection[feature.get(this.vtIdProp)]

          return [
            new Style({
              image: new Circle({
                // radius: selected ? 7 : 10 * feature.get(this.value),
                radius: 5,
                fill: new Fill({ color: 'red' }),
                stroke: new Stroke({
                  color: [255, 0, 0], width: 2,
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
        if (tab !== 'draw') {
          this.drawType = undefined
        }
      },
      onMapClick (evt) {
        let pixel = evt.pixel
        let features = this.$refs.map.$map.getFeaturesAtPixel(pixel)

        if (!features) {
          this.vtSelection = {}
          // force redraw of layer style
          // this.$refs.vtLayer.refresh()
        } else if (features) {
          this.deviceCoordinate = evt.coordinate
          let feature = features[0]
          // let fid = feature.get(this.vtIdProp)

          if (this.vtSelectMode === 'single') {
            this.vtSelection = {}
          }
          // add selected feature to lookup
          // this.vtSelection[fid] = feature

          let properties = feature.getProperties()
          this.title = properties['title']
          this.powerline = properties['powerline']
          this.voltage = properties['voltage']
          this.service_date = properties['service_date']
          this.chem_id = properties['chem_id']
          this.concentrat = properties['concentrat']
          this.timestamp = properties['timestamp']
          this.device_id = properties['timestamp']
          this.job_id = properties['job_id']
          this.amb_temp = properties['amb_temp']
          this.air_pressu = properties['air_pressu']
          this.rel_humid = properties['rel_humid']
          this.precip = properties['precip']
          this.wind_speed = properties['wind_speed']
          this.wind_direc = properties['wind_direc']

          // force redraw of layer style
          // this.$refs.vtLayer.refresh()
        }
      },
      onMapDoneClick () {
        this.vtSelection = {}
        // force redraw of layer style
        // this.$refs.vtLayer.refresh()
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
    background-color: red
    border-radius: 50%;
    display: inline-block

</style>
