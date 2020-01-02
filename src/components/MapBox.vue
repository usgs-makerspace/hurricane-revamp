<template>
  <div
    id="viz_container"
  >
    <LoadingScreen
      v-if="!isInternetExplorer"
      :is-loading="isLoading"
    />

    <div class="header-container">
      <div class="usa-prose">
        <h1 class="title-text">
          {{ title }}{{ titleSuffix }} {{ developmentTier }}
        </h1>
      </div>
    </div>
    <InternetExplorerPage v-if="isInternetExplorer" />
    <div
      v-if="!isInternetExplorer"
      id="mapContainer"
    >
      <MglMap
        id="mapgl"
        :container="container"
        :map-style="mapStyle"
        :zoom="zoom"
        :min-zoom="minZoom"
        :max-zoom="maxZoom"
        :center="center"
        :pitch="pitch"
        :bearing="bearing"
        :pitch-with-rotate="false"
        :drag-rotate="false"
        :touch-zoom-rotate="false"
        @load="onMapLoaded"
      >
        <MglAttributionControl
          position="bottom-right"
          :compact="false"
          custom-attribution="© <a href='https://www.openstreetmap.org/copyright'>OpenStreetMap</a> contributors"
        />

        <MglNavigationControl
          position="top-right"
          :show-compass="false"
        />
        <QuestionControl />
        <MglScaleControl
          position="bottom-right"
          unit="imperial"
        />
        <MglFullscreenControl position="bottom-right" />
        <MglGeolocateControl position="bottom-right" />
      </MglMap>
    </div>
    <!--The next div contains information to show the current zoom level of the map. This will only show on the
          development version of the application. To find the code controlling this, search for 'zoom level display' -->
    <div id="zoom-level-div" />
    <button
      id="fly"
      @click="flyToHurricanePath()"
    >
      Fly
    </button>
    <button
      id="jump"
      @click="jumpTo()"
    >
      Jump
    </button>
    <button
      id="jump2"
      @click="jumpToEnd()"
    >
      Jump to end
    </button>
    <div id="coords" />
  </div>
</template>
<script>
    import LoadingScreen from './LoadingScreen';
    import InternetExplorerPage from "./InternetExplorerPage";
    import QuestionControl from "./QuestionControl";
    import mariaTrackGeoJSON from "../assets/hurricaneTracks/2017_maria";
    import michaelTrackGeoJSON from "../assets/hurricaneTracks/2018_michael";

    import {
        MglMap,
        MglNavigationControl,
        MglGeolocateControl,
        MglFullscreenControl,
        MglScaleControl,
        MglAttributionControl
    } from "vue-mapbox";
    import mapStyles from "../assets/mapStyles/mapStyles";

    export default {
        name: "MapBox",
        components: {
            LoadingScreen,
            InternetExplorerPage,
            MglMap,
            MglNavigationControl,
            MglGeolocateControl,
            MglFullscreenControl,
            MglScaleControl,
            MglAttributionControl,
            QuestionControl
        },
        props: {
            isInternetExplorer: {
                type: Boolean,
                required: true,
                default: false
            }
        },
        data() {
            return {
                title: process.env.VUE_APP_TITLE,
                titleSuffix: process.env.VUE_APP_TITLE_SUFFIX,
                developmentTier: process.env.VUE_APP_TIER,
                mapStyle: mapStyles.style,
                container: "map",
                zoom: 2,
                minZoom: 2,
                maxZoom: 22,
                center: [-95.7129, 37.0902],
                pitch: 60, // tips the map from 0 to 60 degrees
                bearing: 0, // starting rotation of the map from 0 to 360
                isLoading: true,
                hurricaneTrackArray: this.getHurricaneTrackAsArray(),
                hurricaneTrackStyle: this.getHurricaneTrackStyle()

            };
        },
        created() {
           this.map = null; // Once the map is loaded, this will allow us to access the map object in other methods
        },
        methods: {
            getHurricaneTrackAsArray() {
                let hurricaneTrack = [];
                michaelTrackGeoJSON.michaelData.features.forEach(function(feature) {
                    feature.geometry.coordinates.forEach(function (coordinateSet) {
                        hurricaneTrack.push(coordinateSet)
                    });
                });
                return hurricaneTrack;
            },
            getHurricaneTrackStyle() {
                return {
                    'id': 'hurricane track',
                    'type': 'line',
                    'source': 'michaelTrackGeoJSON',
                    'layout': {
                        'visibility': 'visible',
                    },
                    'paint': {
                        'line-color': 'red',
                        'line-width': {
                            'base': 2.55,
                            'stops': [
                                [0, 4],
                                [20, 8]
                            ]
                        }
                    }
                }
            },
            addZoomLevelIndicator() {
                document.getElementById("zoom-level-div").innerHTML = 'Current Zoom Level (listed for development purposes): ' + this.map.getZoom() ;
            },
            addHurricanePathToMap(hurricaneTrackGeoJSON) {
                this.map.addSource("michaelTrackGeoJSON", {
                    "type": "geojson",
                    "data": hurricaneTrackGeoJSON
                });
            },
            jumpTo() {
                let currentHurricaneTrack = this.hurricaneTrackArray;
                let map = this.map;

                let fly = function() {
                    map.flyTo({
                        center: currentHurricaneTrack[0],
                        zoom: 5,
                        bearing: 20,
                        speed: 0.12, // make the flying slow
                        curve: 1, // change the speed at which it zooms out
                    });
                };

                let jump = function() {
                    currentHurricaneTrack.forEach(function(obj,index) {
                        setTimeout(function(){
                            map.jumpTo({center: currentHurricaneTrack[index], zoom: 5})
                            console.log('count ', index);
                        }, index * 1000);
                    });
                };
                // fly to the starting point
                fly();
                // use a promise to delay the start of the jumpTo action
                let promise1 = new Promise(function(resolve, reject) {
                    setTimeout(function() {
                        resolve('time up');
                    }, 7000);
                });

                promise1.then(function(value) {
                    jump();
                });
            },
            jumpToEnd() {
                let endIndex = this.hurricaneTrackArray.length -1;
                console.log('ran jump to end. this b end index ', endIndex)
                this.map.jumpTo({center: this.hurricaneTrackArray[endIndex]})
            },
            flyToHurricanePath() {
                let hurricaneTrack = this.hurricaneTrackArray;
                const degreesToRadians = function(degrees) {
                    return degrees * Math.PI / 180;
                };
                const distanceInKmBetweenEarthCoordinates = function(lat1, lon1, lat2, lon2) {
                    let earthRadiusKm = 6371;
                    let dLat = degreesToRadians(lat2-lat1);
                    let dLon = degreesToRadians(lon2-lon1);
                    lat1 = degreesToRadians(lat1);
                    lat2 = degreesToRadians(lat2);
                    let a = Math.sin(dLat/2) * Math.sin(dLat/2) +
                            Math.sin(dLon/2) * Math.sin(dLon/2) * Math.cos(lat1) * Math.cos(lat2);
                    let c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
                    return earthRadiusKm * c;
                };
                let coorDiv = document.getElementById("coords");
                let map = this.map;

                    let i = 0;
                    let mapDistance = null;
                    let distanceAdjustmentMultiplyer = 1;
                    let baseDurationOfFlyToAction = 1000;
                    let intervalId = setInterval(function(){
                        if(i === hurricaneTrack.length - 1){
                            clearInterval(intervalId);
                        }
                        coorDiv.innerHTML = 'the current coordinates ' + hurricaneTrack[i][0] + ', ' + hurricaneTrack[i][1]
                        if (i < hurricaneTrack.length - 1) {
                            mapDistance = distanceInKmBetweenEarthCoordinates(hurricaneTrack[i][0], hurricaneTrack[i][1], hurricaneTrack[i+1][0], hurricaneTrack[i+1][1]);
                        }
                        if (mapDistance !== 0) {
                            distanceAdjustmentMultiplyer = baseDurationOfFlyToAction/mapDistance;
                            console.log('this is the multiplier ', distanceAdjustmentMultiplyer)
                        }
                        coorDiv.innerHTML = 'the current coordinates are ' + hurricaneTrack[i][0] + ', ' + hurricaneTrack[i][1] + ', and the distance between is ' + mapDistance + ' kilometers.'

                        map.flyTo({
                            center: [hurricaneTrack[i][0], hurricaneTrack[i][1]],
                            zoom: 7,
                            bearing: 0,
                            speed: 0.12, // make the flying slow
                            curve: 1, // change the speed at which it zooms out
                        });
                        i++;
                        console.log('duration ' + baseDurationOfFlyToAction/distanceAdjustmentMultiplyer)
                        // }, baseDurationOfFlyToAction * distanceAdjustmentMultiplyer);
                    }, baseDurationOfFlyToAction);

            },

            onMapLoaded(event) {
                this.map = event.map; // This gives us access to the map as an object but only after the map has loaded.
                this.map.resize(); // This cures the mysterious whitespace that appears above the footer is was caused by the 'official' banner at the top.
                this.map.touchZoomRotate.enable({ around: 'center' }); // Add pinch to zoom for touch devices.
                this.map.touchZoomRotate.disableRotation(); // Disable the rotation functionality, but keep pinch to zoom.
                this.map.fitBounds([[-125.3321, 23.8991], [-65.7421, 49.4325]]); // Once map is loaded, zoom in a bit more so that the map neatly fills the screen.
                this.$store.map = event.map; // Add the map to the Vuex store so that we can use it in other components.
                // Pause the code here to make sure the fitbounds has time to finish before fade away of loading screen.
                setTimeout(() => { this.isLoading = false; }, 200);
                // Add the current zoom level display. The zoom level should only show in 'development' versions of the application.
                process.env.VUE_APP_ADD_ZOOM_LEVEL_DISPLAY === 'true' ? this.map.on("zoomend", this.addZoomLevelIndicator) : null;

                this.addHurricanePathToMap(michaelTrackGeoJSON.michaelData);
                this.map.addLayer(this.getHurricaneTrackStyle());
                var frameCount = 6;
                var currentImage = 0;
                
                function getPath() {
                  return require('../images/radar/n0r-t'+currentImage+'.png') 
                }
                this.map.addSource('radar', {
                type: 'image',
                url: getPath(),
                  coordinates: [
                    [-93.8671875, 48.6909603], //top left
                    [-65.7421875, 48.6909603], //top right
                    [-65.7421875, 25.0059728], //bottom right
                    [-93.8671875, 25.0059728]  //bottom left
                  ]
                });
                this.map.addLayer({
                  id: 'radar-layer',
                  'type': 'raster',
                  'source': 'radar',
                  'paint': {
                      'raster-fade-duration': 0
                  }
                });
                
                setInterval(function() {
                  currentImage = (currentImage + 1) % frameCount;
                  map.getSource('radar').updateImage({ url: getPath() });
                }, 200);
            }
        }
    };
</script>

<style scoped lang="scss">
  @import "~mapbox-gl/dist/mapbox-gl.css";
  $color: #fff;
  $blue: #4574a3;
  $border: 1px solid #fff;
  $borderGray: 1px solid rgb(100, 100, 100);

  #overlay {
    position: fixed; /* Sit on top of the page content */
    display: none; /* Hidden by default */
    width: 100%; /* Full width (cover the whole page) */
    height: 100%; /* Full height (cover the whole page) */
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background-color: rgba(0,0,0,0.5); /* Black background with opacity */
    z-index: 2; /* Specify a stack order in case you're using a different order for other elements */
    cursor: pointer; /* Add a pointer on hover */
  }

  .usa-prose {
    border-bottom: $borderGray;
    display: flex;
    h1 {
      font-size: 1rem;
      margin-left: 10px;
      flex: 1;
    }
    a{
      margin: 0;
      display: block;
    }
  }

  .title-text {
    margin-left: 1.5rem;
    padding: 0.25rem;
  }

  #mapContainer {
    position: relative;
    height: 80vh;
    min-height: 550px;
    display: flex;
    flex-direction: column;
  }

  @media screen and (min-width: 600px) and (min-height: 850px) {
    #viz_container {
      flex: 1;
      display: flex;
      flex-direction: column;
    }

    #mapContainer {
      flex: 1;
      height: auto;
    }
  }
</style>


