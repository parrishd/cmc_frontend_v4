<template>
  <div class="map-container">
    <!-- MapBox GL JS instance -->

    <div
      :id="
        Object.prototype.hasOwnProperty.call(mapOptions, 'container')
          ? mapOptions.container
          : 'map'
      "
    ></div>
    <div v-if="showLegend">
      <!--
      <div class="map-overlay">
        <q-card class="my-card">
          <q-card-section>
            <q-list>
              <p>Sampling Status</p>

              <p>
                <span class="dot-purple"></span>
                &nbsp;Current
              </p>

              <p>
                <span class="dot-gray"></span>
                &nbsp;Historic
              </p>
            </q-list>
          </q-card-section>
        </q-card>
      </div>
      -->

      <!-- toggle raster or vector map style. Check uncheck to show polygon data-->
      <div class="legend-style">
        <q-card class="my-card">
          <q-card-section>
            <div style="font-size: 16px" large>Data Layers</div>
            <q-checkbox v-model="showStations" class="q-mt-sm">
              {{ selectedDataType }}</q-checkbox
            >
            <!--add padding to left of this div-->
            <div style="padding-left: 30px; padding-top: 5px">
              <p>
                <span
                  :class="{
                    'dot-purple': selectedDataType == 'Water Quality',
                    'dot-orange': selectedDataType != 'Water Quality',
                  }"
                ></span>
                &nbsp;Active Sites
              </p>

              <p>
                <span class="dot-gray"></span>
                &nbsp;Inactive Sites
              </p>
            </div>
            <div class="row" v-show="selectedGeoType == 'Watershed'">
              <q-checkbox v-model="showPolygon1"> Watersheds</q-checkbox>
            </div>
            <div class="row" v-show="selectedGeoType == 'Watershed'">
              <q-checkbox v-model="showPolygon2"> SubWatersheds</q-checkbox>
            </div>
            <div class="row" v-show="selectedGeoType !== 'Watershed'">
              <q-checkbox v-model="showPolygon1"> States</q-checkbox>
            </div>
            <div class="row" v-show="selectedGeoType !== 'Watershed'">
              <q-checkbox v-model="showPolygon2"> Counties</q-checkbox>
            </div>
          </q-card-section>
          <q-card-section>
            <div style="font-size: 16px" large>Basemap</div>
            <q-option-group
              class="q-mt-sm"
              v-model="mapStyle"
              :options="mapStyleOptions"
              color="primary"
            ></q-option-group>
          </q-card-section>
        </q-card>
      </div>
    </div>
  </div>
</template>

<script setup>
import mapboxgl from "mapbox-gl";
import counties from "../assets/spatial/counties_simple.json";
import states from "../assets/spatial/states_simple.json";
import huc8 from "../assets/spatial/huc8_simple.json";
import huc12 from "../assets/spatial/huc12_simple.json";
import { onMounted, reactive, ref, watch, toRefs, onUnmounted } from "vue";
import { data } from "autoprefixer";

// *** Component Props ***
const props = defineProps([
  "mapData",
  "mapOptions",
  "showLegend",
  "collapsed",
  "selectedDataType",
  "selectedGeoType",
]);
const { mapData } = toRefs(props);
const { mapOptions } = toRefs(props);
const { showLegend } = toRefs(props);
const { collapsed } = toRefs(props);
const { selectedDataType } = toRefs(props);
const { selectedGeoType } = toRefs(props);

const emit = defineEmits(["selectedStation"]);

// *** Variables and constants ***

// instance of MapBox GL JS
let map = reactive({});

// shows hides polygons that represent 50 states per foo geojson file
let showPolygon1 = ref(true);
let showPolygon2 = ref(false);
let showStations = ref(true);

// used to persist the polygons layer pop up in case we toggle data type
//let isShowingPolygonPopUp = ref(false);

// group of options to select to show map as aerial (raster) vs vector street map
let mapStyle = ref("vector");
const mapStyleOptions = ref([
  {
    label: "Street Map",
    value: "vector",
  },
  {
    label: "Aerial",
    value: "raster",
  },
]);

let stationLayersIdsMap = new Map();

// locs static foo data for demo on markers
let locs = reactive({
  type: "FeatureCollection",
  features: [],
});

// *** Watch properties ***
watch(mapData, () => {
  if (mapData.value.length > 0) {
    removePopUps();
    getStations();
  }
});

if (showLegend.value === true) {
  watch(collapsed, () => {
    console.log("collapsed: " + collapsed.value);

    setTimeout(() => (map.resize(), 50));
  });
}

watch(showPolygon1, () => {
  console.log("showpolygon1: " + showPolygon1.value);
  togglePolygon1();
});

watch(showPolygon2, () => {
  console.log("showpolygon2: " + showPolygon2.value);
  togglePolygon2();
});

watch(showStations, () => {
  console.log("showStations: " + showStations.value);
  toggleStations();
});

watch(selectedGeoType, () => {
  console.log("geoType: " + selectedGeoType.value);
  togglePolygon1();
  togglePolygon2();
});
watch(selectedDataType, () => {
  console.log("geoType: " + selectedGeoType.value);
  toggleStations();
});

// https://github.com/mapbox/mapbox-gl-js/issues/8660
// https://github.com/mapbox/mapbox-gl-js/issues/2267
// links above is for issues with layers and sources when toggling map styles
watch(mapStyle, () => {
  console.log('mapStyle.value " + mapStyle.value');
  if (mapStyle.value) {
    if (mapStyle.value === "raster") {
      map.setStyle("mapbox://styles/mapbox/satellite-streets-v12");
    } else {
      map.setStyle("mapbox://styles/mapbox/streets-v12");
    }
  }
});

//  *** Functions ***

// used to populate static data from foo json files
const getStations = () => {
  console.log("getStations");
  console.log(mapData.value);
  if (mapData.value) {
    let locations = {
      type: "FeatureCollection",
      features: [],
    };
    mapData.value.forEach((s) => {
      let station = {
        type: "Feature",
        properties: {
          selected: false,
          code: s.StationCode,
          groupNames: s.GroupNames,
          id: s.StationId,
          latitude: s.Lat,
          longitude: s.Long,
          status: s.Status,
          startDate: s.formattedStartDate,
          endDate: s.formattedEndDate,
        },
        geometry: {
          type: "Point",
          coordinates: [s.Long, s.Lat],
        },
        id: s.id,
      };
      locations.features.push(station);
    });
    locs = locations;

    createMap();
  }
};

const createMap = () => {
  mapboxgl.accessToken =
    "pk.eyJ1IjoibXlha2F2ZW5rYSIsImEiOiJjbDlxMDJrNmcwMmE2M3dxeDYyZWE0OWQ0In0.dKzXgJu-ZUH3epnFzxvllg";
  map = new mapboxgl.Map(mapOptions.value);
  map.addControl(new mapboxgl.NavigationControl());

  //this was causing the map to flash, removing for now. There may be a better way to handle this. Leave this commented code here for now.
  //map.on("load", () => {
  //  addStatesPolygons();
  //  updateMarkersOnMap();
  //});

  map.on("style.load", () => {
    console.log("A styledata event occurred.");

    if (showPolygon1.value === true) {
      // re-add polygons in case check box was still on
      //console.log("skip polygons for now");
      addPolygon1();
    }
    if (showPolygon2.value === true) {
      // re-add polygons in case check box was still on
      //console.log("skip polygons for now");
      addPolygon2();
    }
    // re-add all the markers per whatever data type is selected
    updateMarkersOnMap();
  });
};

const togglePolygon1 = () => {
  removePopUps();
  if (showPolygon1.value === true) {
    addPolygon1();
    updateMarkersOnMap();
  } else {
    console.log("should be hiding the polygons");
    removePolygon1();
    removePopUps();
  }
};
const togglePolygon2 = () => {
  removePopUps();
  if (showPolygon2.value === true) {
    addPolygon2();
    updateMarkersOnMap();
  } else {
    console.log("should be hiding the polygons");
    removePolygon2();
    removePopUps();
  }
};

const toggleStations = () => {
  removePopUps();
  if (showStations.value === true) {
    console.log("should be showing the stationss");
    setupStationsOnMap();
  } else {
    removeStations();
  }
};

// removes the layers associated with geojsons states polygons and removes source after
const removePolygon1 = () => {
  if (map.getSource("polygon1")) {
    const layer = "polygon1";
    const layerForPolygon = "polygon1-layer";
    const layerForPolygonOutline = "polygon1-layer-outline";

    if (map.getLayer(layerForPolygonOutline)) {
      map.removeLayer(layerForPolygonOutline);
    }

    if (map.getLayer(layerForPolygon)) {
      // map.off('click', 'states-layer', onClickPolygon);
      // map.off('mouseenter', 'states-layer')
      // map.off('mouseleave', 'states-layer')
      map.removeLayer(layerForPolygon);
    }

    // finally remove the source after removing layers and callbacks
    map.removeSource(layer);
  }
};

// removes the layers associated with geojsons states polygons and removes source after
const removePolygon2 = () => {
  if (map.getSource("polygon2")) {
    const layer = "polygon2";
    const layerForPolygon = "polygon2-layer";
    const layerForPolygonOutline = "polygon2-layer-outline";

    if (map.getLayer(layerForPolygonOutline)) {
      map.removeLayer(layerForPolygonOutline);
    }

    if (map.getLayer(layerForPolygon)) {
      // map.off('click', 'states-layer', onClickPolygon);
      // map.off('mouseenter', 'states-layer')
      // map.off('mouseleave', 'states-layer')
      map.removeLayer(layerForPolygon);
    }

    // finally remove the source after removing layers and callbacks
    map.removeSource(layer);
  }
};

// Add a data source containing GeoJSON data.
// https://docs.mapbox.com/mapbox-gl-js/example/polygon-popup-on-click/
const addPolygon1 = () => {
  console.log("addPolygon1");
  console.log("window.location.origin: " + window.location.origin);
  //console.log(window.location.origin);
  //console.log(counties);
  //console.log(counties.value);
  //console.log(huc8.value);

  let polygons = states;
  if (selectedGeoType.value === "Watershed") {
    polygons = huc8;
  }
  if (map.getSource("polygon1")) {
    removePolygon1();
  }
  map.addSource("polygon1", {
    type: "geojson",
    data: polygons,
  });

  // Add a clear layer showing that will be used for on click events
  map.addLayer({
    id: "polygon1-layer",
    type: "fill",
    source: "polygon1",
    paint: {
      "fill-color": "rgba(0, 0, 0, 0.0)",
      // 'fill-outline-color': 'rgba(200, 100, 240, 1)',
    },
  });

  // Add an outline layer that will be visible and will outline geojson states polygons
  map.addLayer({
    id: "polygon1-layer-outline",
    type: "line",
    source: "polygon1",
    paint: {
      "line-color": "rgba(7, 153, 153, 0.9)",
      "line-width": 3,
    },
  });
  // When a click event occurs on a feature in the states layer,
  // open a popup at the location of the click, with description
  // HTML from the click event's properties.
  map.on("mouseenter", "polygon1-layer", onMouseEnterEvent);
  map.on("mouseleave", "polygon1-layer", onMouseLeaveEvent);
  map.on("click", "polygon1-layer", (e) => {
    map.fire("closeAllStationsPopups");
    map.fire("closeAllPolygonPopups");
    console.log("On click polygon event");
    console.log(e);
    console.log(e.features[0]);
    let polyName = e.features[0].properties.STATE_NAME;
    if (selectedGeoType.value === "Watershed") {
      polyName = e.features[0].properties.ACTNAME;
    }
    const popup = new mapboxgl.Popup()
      .setLngLat(e.lngLat)
      .setHTML(polyName)
      .addTo(map);

    // flag used to persist this pop up if data type changes as those pop ups need to be removed.
    // isShowingPolygonPopUp.value = true;
    // popup.on("close", () => {
    //   console.log("Why tho here? we set it false on close..");
    //   isShowingPolygonPopUp.value = false;
    // });
    map.on("closeAllPolygonPopups", () => {
      popup.remove();
    });
    //map.fire("closeAllPolygonPopups");
  });
};
// Add a data source containing GeoJSON data.
// https://docs.mapbox.com/mapbox-gl-js/example/polygon-popup-on-click/
const addPolygon2 = () => {
  console.log("addPolygon2");
  console.log("window.location.origin: " + window.location.origin);
  //console.log(window.location.origin);
  //console.log(counties);
  //console.log(counties.value);
  //console.log(huc8.value);

  let polygons = counties;
  if (selectedGeoType.value === "Watershed") {
    polygons = huc12;
  }
  console.log("addPolygon2");
  console.log(map.getSource("polygon2"));
  if (map.getSource("polygon2")) {
    ("removing polygon2 layer and source");
    removePolygon2();
  }
  map.addSource("polygon2", {
    type: "geojson",
    data: polygons,
  });

  // Add a clear layer showing that will be used for on click events
  map.addLayer({
    id: "polygon2-layer",
    type: "fill",
    source: "polygon2",
    paint: {
      "fill-color": "rgba(0, 0, 0, 0.0)",
      // 'fill-outline-color': 'rgba(200, 100, 240, 1)',
    },
  });

  // Add an outline layer that will be visible and will outline geojson states polygons
  map.addLayer({
    id: "polygon2-layer-outline",
    type: "line",
    source: "polygon2",
    paint: {
      "line-color": "rgba(7, 153, 153, 0.9)",
      "line-width": 1,
    },
  });

  // When a click event occurs on a feature in the states layer,
  // open a popup at the location of the click, with description
  // HTML from the click event's properties.
  map.on("mouseenter", "polygon2-layer", onMouseEnterEvent);
  map.on("mouseleave", "polygon2-layer", onMouseLeaveEvent);
  map.on("click", "polygon2-layer", (e) => {
    map.fire("closeAllStationsPopups");
    map.fire("closeAllPolygonPopups");
    console.log("On click polygon event");
    console.log(e);
    console.log(e.features[0]);
    let polyName = e.features[0].properties.NAMELSAD20;
    if (selectedGeoType.value === "Watershed") {
      polyName = e.features[0].properties.NAME;
    }
    const popup = new mapboxgl.Popup()
      .setLngLat(e.lngLat)
      .setHTML(polyName)
      .addTo(map);

    // flag used to persist this pop up if data type changes as those pop ups need to be removed.
    // isShowingPolygonPopUp.value = true;
    // popup.on("close", () => {
    //   console.log("Why tho here? we set it false on close..");
    //   isShowingPolygonPopUp.value = false;
    // });
    map.on("closeAllPolygonPopups", () => {
      popup.remove();
    });
    //map.fire("closeAllPolygonPopups");
  });
};
// Change the cursor to a pointer when
// the mouse is over the layer.
const onMouseEnterEvent = () => {
  map.getCanvas().style.cursor = "pointer";
};

// Change the cursor back to a pointer
// when it leaves the layer.
const onMouseLeaveEvent = () => {
  map.getCanvas().style.cursor = "";
};

// When a click event occurs on a feature in the states layer,
// open a popup at the location of the click, with description
// HTML from the click event's properties.
// const onClickPolygon = (e) => {
//   console.log(e)
//   console.log('On click polygon event');
//
//   removePopUps();
//   const popup = new mapboxgl.Popup()
//     .setLngLat(e.lngLat)
//     .setHTML(e.features[0].properties.name)
//     .addTo(map);
//
//   // flag used to persist this pop up if data type changes as those pop ups need to be removed.
//   console.log("Why tho here?");
//   isShowingPolygonPopUp.value = true;
//   popup.on('close', () => {
//     isShowingPolygonPopUp.value = false
//   })
//   console.log("yoyoyoy testing foo");
// }

// checks per data type id and then loops over the data to find and remove each layer prior to removing the source
// used to clear out map on toggle of data source
const removeStations = () => {
  if (map.getSource("stations")) {
    map.removeLayer("places");
    map.removeSource("stations"); // remove
  }
};

function stationDetailsClicked(station) {
  console.log("stationDetailsClicked");
  console.log(station);
  emit("selectedStation", station);
}

const setupStationsOnMap = () => {
  console.log("setup stations on map");
  console.log(locs);

  console.log("time stamp map stations 1: " + new Date().getTime());
  if (locs.features.length === 0) {
    return;
  }
  if (map.getSource("stations")) {
    removeStations();
  }
  map.addSource("stations", {
    type: "geojson",
    data: locs,
  });
  map.getSource("stations").setData(locs);
  console.log("time stamp map stations 2: " + new Date().getTime());
  let minLong = 0;
  let maxLong = -100;
  let minLat = 40;
  let maxLat = 30;
  locs.features.forEach((f, index) => {
    //console.log(f.properties["latitude"], f.properties["longitude"]);
    if (f.properties["longitude"] < minLong) {
      minLong = f.properties["longitude"];
    }
    if (f.properties["latitude"] < minLat) {
      minLat = f.properties["latitude"];
    }
    if (f.properties["longitude"] > maxLong) {
      maxLong = f.properties["longitude"];
    }
    if (f.properties["latitude"] > maxLat) {
      maxLat = f.properties["latitude"];
    }
  });
  //console log time stamp
  console.log("time stamp map stations 3: " + new Date().getTime());

  //if (showLegend.value === true) {
  let centerLat = (minLat + maxLat) / 2; // + 1.8;
  let centerLong = (minLong + maxLong) / 2;
  let centerCoordinates = { lng: centerLong, lat: centerLat };
  map.setCenter(centerCoordinates);
  console.log("centerCoordinates: " + centerLat + " " + centerLong);
  let locColor = "#990799";
  if (selectedDataType.value === "Benthic Macroinvertebrates") {
    locColor = "#8b6508";
  }
  map.addLayer({
    id: "places",
    type: "circle",
    source: "stations",
    paint: {
      // 'circle-radius': 7,
      "circle-color": [
        "match",
        ["get", "status"], // get the property
        "Historic",
        "#5A5A5A", // if 'GP' then yellow
        "Current",
        locColor, // if 'XX' then black
        "white",
      ], // white otherwise
      "circle-radius": [
        "case",
        ["boolean", ["feature-state", "hover"], false],
        22,
        7,
      ],
      "circle-stroke-color": "white",
      "circle-stroke-width": 1,
      "circle-opacity": 0.5,
    },
  });
  console.log("time stamp map stations 4: " + new Date().getTime());
  // When a click event occurs on a feature in the places layer, open a popup at the
  // location of the feature, with description HTML from its properties.
  map.on("click", "places", (e) => {
    // Copy coordinates array.
    map.fire("closeAllPolygonPopups");
    map.fire("closeAllStationPopups");
    console.log(e);
    console.log(e.features[0].properties.groupnames);
    const coordinates = e.features[0].geometry.coordinates.slice();
    const description = e.features[0].properties.id;
    const feature = e.features[0];

    const stationDetails = {
      code: feature.properties.code,
      groupNames: feature.properties.groupNames,
      id: feature.properties.id,
      latitude: feature.properties.latitude,
      longitude: feature.properties.longitude,
      status: feature.properties.status,
      startDate: feature.properties.startDate,
      endDate: feature.properties.endDate,
    };
    console.log(stationDetails);
    console.log(feature.properties);
    console.log(feature.properties.code);
    console.log(feature.properties.groupNames);
    const innerHtmlContent = `
    <div style="padding: 10px 20px 10px 20px">
      <div style="color: #075C7A; font-size: 1.95em; font-weight: 700;">
        Station
      </div>
      <div style="margin-top: 10px; padding-left: 20px; font-size: 1.35em; font-weight: 400;">
        ${feature.properties.code}
      </div>
      <div style="margin-top:10px; color: #075C7A; font-size: 1.95em; font-weight: 700;">
        Monitored By
      </div>
      <div style="margin-top: 10px; padding-left: 20px; font-size: 1.35em; font-weight: 400;">
        ${feature.properties.groupNames}
      </div>
      <div style="margin-top: 10px">

      </div>
    </div>
    `;
    const divElement = document.createElement("div");
    const assignBtn = document.createElement("div");
    assignBtn.innerHTML = ` <button style="background-color: #075C7A;cursor:pointer; color: white; padding: 8px 10px 8px 10px; border: none; border-radius: 5px;">
              View Station Details
            </button>`;
    divElement.innerHTML = innerHtmlContent;
    divElement.appendChild(assignBtn);

    assignBtn.addEventListener("click", () => {
      console.log("Button clicked");

      stationDetailsClicked(stationDetails);
    });

    const popup = new mapboxgl.Popup({
      offset: [0, 0],
      maxWidth: "400px",
    })
      .setLngLat(feature.geometry.coordinates)
      .setDOMContent(divElement);

    // Add a custom event listener to the map
    map.on("closeAllStationPopups", () => {
      popup.remove();
    });
    popup.addTo(map);
    //setupMouseEventListeners(layerID);

    //new mapboxgl.Popup().setLngLat(coordinates).setHTML(divElement).addTo(map);
  });
  console.log("time stamp map stations 5: " + new Date().getTime());
  //}
};

const updateMarkersOnMap = () => {
  console.log("updateMarkersOnMap");
  removeStations();

  console.log("*** data setup after update***");
  setupStationsOnMap();
};

const setupMouseEventListeners = (layerID) => {
  map.on("mouseenter", layerID, function () {
    map.getCanvas().style.cursor = "pointer";
  });
  map.on("mouseleave", layerID, function () {
    map.getCanvas().style.cursor = "";
  });
};

const removePopUps = () => {
  map.fire("closeAllPolygonPopups");
  map.fire("closeAllStationPopups");
};

onMounted(() => {
  getStations();
});
onUnmounted(() => {
  console.log("unmounted");
  map.remove();
});
</script>

<style scoped>
.map-container {
  position: relative;
}

.legend-style {
  font: 12px/20px "Helvetica Neue", Arial, Helvetica, sans-serif;
  position: absolute;
  width: 220px;
  top: 5px; /* 154px if want to adjust below data type widget */
  left: 0px;
  padding: 4px;
}

.data-type-widget {
  font: 12px/20px "Helvetica Neue", Arial, Helvetica, sans-serif;
  position: absolute;
  width: 180px;
  top: 0px;
  left: 0px;
  padding: 4px;
}

.map-overlay {
  font: 12px/20px "Helvetica Neue", Arial, Helvetica, sans-serif;
  position: absolute;
  width: 180px;
  bottom: 0px;
  left: 0px;
  padding: 4px;
}

#map {
  height: 800px; /* 82vh if want %height */
}

#map1 {
  height: 500px;
}

.dot-green {
  height: 15px;
  width: 15px;
  background-color: #21ba45;
  border-radius: 50%;
  display: inline-block;
  vertical-align: middle;
}

.dot-red {
  height: 15px;
  width: 15px;
  background-color: #e91e63;
  border-radius: 50%;
  display: inline-block;
  vertical-align: middle;
}
.dot-blue-green {
  height: 15px;
  width: 15px;
  background-color: #0c9482;
  border-radius: 50%;
  display: inline-block;
  vertical-align: middle;
}

.dot-yellow {
  height: 15px;
  width: 15px;
  background-color: #1b5e20;
  border-radius: 50%;
  display: inline-block;
  vertical-align: middle;
}

.dot-orange {
  height: 15px;
  width: 15px;
  background-color: #8b6508;
  border-radius: 50%;
  display: inline-block;
  vertical-align: middle;
}

.dot-purple {
  height: 15px;
  width: 15px;
  background-color: #990799;
  border-radius: 50%;
  display: inline-block;
  vertical-align: middle;
}

.dot-gray {
  height: 15px;
  width: 15px;
  background-color: #5a5a5a;
  border-radius: 50%;
  display: inline-block;
  vertical-align: middle;
}
</style>
