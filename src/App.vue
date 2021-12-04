<template>
    <div id="root">
        <div id="map"></div>
        <div class="overlay top left flex-col gap">

            <Button
            @click="clearMarkers"
            class="bg-warning"
            text="Clear markers"
            optText="(for performance, this won't erase your generated locations)"
            title="Clear markers"
            />
        </div>

        <div class="overlay top right flex-col gap">
            <div class="settings">
                <h4 class="center">Settings</h4>
                <Checkbox v-model:checked="settings.rejectUnofficial" label="Reject unofficial" />
                <hr />

                <div class="flex space-between mb-2">
                    <label>From</label>
                    <input type="month" v-model="settings.fromDate" min="2007-01" :max="dateToday" />
                </div>
                <div class="flex space-between">
                    <label>To</label>
                    <input type="month" v-model="settings.toDate" :max="dateToday" />
                </div>
                <hr />
                <div class="flex space-between">
                    <input class="center mb-2" type="text" id="panoid" placeholder="Pano ID" />
                </div>
                <Button
                @click="searchPano"
                class="bordered-success center"
                text="Check Pano ID"
                title="Check Pano ID"
                />
                <hr/>
                <div class="space-between">
                    <Checkbox v-model:checked="settings.cluster" v-on:change="checkClusters" label="Cluster markers" />
                    <small>For lag reduction.</small>
                </div>

            </div>
        </div>

        <div class="overlay export bottom right">
            <h4 class="center mb-2">Export selection to</h4>
            <div class="flex gap">
                <Button @click="exportToJSON()" class="bordered-success" text="JSON" title="Export as JSON" />
                <Button @click="exportToCSV()" class="bordered-success" text="CSV" title="Export as CSV"/>
            </div>
        </div>
    </div>
</template>


<script setup>
import { onMounted, ref, reactive, computed } from "vue";
import Button from "./components/Button.vue";
import Checkbox from "./components/Checkbox.vue";

import L from "leaflet";
import "leaflet/dist/leaflet.css";
import "leaflet-draw/dist/leaflet.draw.js";
import "leaflet-draw/dist/leaflet.draw.css";

import "leaflet.markercluster/dist/leaflet.markercluster.js";
import "leaflet.markercluster/dist/MarkerCluster.css";
import "leaflet.markercluster/dist/MarkerCluster.Default.css";

import "leaflet.markercluster.freezable/dist/leaflet.markercluster.freezable.js";

import booleanPointInPolygon from "@turf/boolean-point-in-polygon";

const dateToday = new Date().getFullYear() + "-" + (new Date().getMonth() + 1);

const settings = reactive({
	rejectUnofficial: true,
	fromDate: "2009-01",
	toDate: dateToday,
    cluster: false
});

const checkedPanos = new Set();
let panos = {};
let map;
const SV = new google.maps.StreetViewService();

const markerLayer = L.markerClusterGroup({
    maxClusterRadius: 25,
    disableClusteringAtZoom: 20
});

markerLayer.disableClustering();

const customPolygonsLayer = new L.FeatureGroup();
const drawControl = new L.Control.Draw({
    position: "bottomleft",
    draw: {
        polyline: false,
        marker: false,
        circlemarker: false,
        circle: false,
        polygon: {
            allowIntersection: false,
            drawError: {
                color: "#e1e100",
                message: "<strong>Polygon draw does not allow intersections!<strong> (allowIntersection: false)"
            },
            shapeOptions: { color: "#5d8ce3" }
        },
        rectangle: { shapeOptions: { color: "#5d8ce3" } }
    },
    edit: { featureGroup: customPolygonsLayer }
});

function searchPano() {
    checkPano(document.getElementById("panoid").value);
}

function checkClusters() {
    if(settings.cluster) markerLayer.enableClustering();
    else markerLayer.disableClustering();
}

function coordsInBounds(coords) {
    if(!Object.keys(customPolygonsLayer._layers).length) return true;
    for (var bounds of Object.values(customPolygonsLayer._layers)) {
        if(!booleanPointInPolygon([coords[1], coords[0]], bounds.toGeoJSON())) return false;
    }
    return true;
}

function checkCoords(lat, lng) {
    SV.getPanorama(
        {
            location: { lat: lat, lng: lng },
            preference: google.maps.StreetViewPreference.NEAREST
        },
        (res, status) => {
            if (status === "ZERO_RESULTS") return;
            if (status !== "OK") {
                console.error("Error at " + lat + " , " + lng + " : " + status);
                return;
            } // no result
            checkPano(res.location.pano);

            for (var g of res.time) checkPano(g.pano);
            for (var h of res.links) checkPano(h.pano);
        }
    );
}

function checkPano(id) {
    if (settings.rejectUnofficial && id.length != 22) return;
    if (checkedPanos.has(id)) return;
    else checkedPanos.add(id);

    SV.getPanorama({ pano: id }, (res, status) => {
        if (status === "UNKNOWN_ERROR") {
            console.warn("Unknown error with " + id + " , trying again.");
            checkedPanos.delete(id);
            checkPano(id);
            return;
        }
        if (status !== "OK") {
            // no result?
            console.error("Uncaught error with Pano ID result?: " + id + " , " + status);
            return;
        }
        if(!coordsInBounds([res.location.latLng.lat(),res.location.latLng.lng()])) {
            checkedPanos.delete(id);
            return;
        }
        if (Date.parse(res.imageDate) < Date.parse(settings.fromDate) || Date.parse(res.imageDate) > Date.parse(settings.toDate)) {
            checkedPanos.delete(id);
            return;
        }
        panos[res.location.pano] = {
            lat: res.location.latLng.lat(),
            lng: res.location.latLng.lng(),
            date: res.imageDate
        };

        L.marker([res.location.latLng.lat(), res.location.latLng.lng()], {
            title: id,
            opacity: 0.75
        })
        .on("click", () => {
            window.open(
                `https://www.google.com/maps/@?api=1&map_action=pano&pano=${
                    res.location.pano
                }
                ${location.heading ? "&heading=" + location.heading : ""}
                ${location.pitch ? "&pitch=" + location.pitch : ""}`,
                "_blank"
            );
        })
        .addTo(markerLayer);

        checkCoords(res.location.latLng.lat(), res.location.latLng.lng());
        for (var g of res.time) checkPano(g.pano);
        for (var h of res.links) checkPano(h.pano);


    });
}

function exportToJSON() {
    const dataUri = "data:application/json;charset=utf-8," + encodeURIComponent(JSON.stringify(panos));
    const fileName = `Generated map.json`;
    const linkElement = document.createElement("a");
    linkElement.href = dataUri;
    linkElement.download = fileName;
    linkElement.click();
}

function exportToCSV() {
    let csv = "";
	for(var h of Object.values(panos)) {
		csv+=h.lat+","+h.lng+"\n";
	}
    const dataUri = "data:application/json;charset=utf-8," + encodeURIComponent(csv);
    const fileName = `Generated map.csv`;
    const linkElement = document.createElement("a");
    linkElement.href = dataUri;
    linkElement.download = fileName;
    linkElement.click();
}

function customPolygonStyle() {
    return {
        weight: 2,
        opacity: 1,
        color: getRandomColor(),
        fillOpacity: 0,
    };
}

function getRandomColor() {
    const red = Math.floor(((1 + Math.random()) * 256) / 2);
    const green = Math.floor(((1 + Math.random()) * 256) / 2);
    const blue = Math.floor(((1 + Math.random()) * 256) / 2);
    return "rgb(" + red + ", " + green + ", " + blue + ")";
}

function clearMarkers() {
	markerLayer.clearLayers();
}

function onClick(e) {
    //checkCoords(e.latlng.lat, e.latlng.lng);
}

onMounted(function() {
    var map = L.map("map", {
        attributionControl: false,
        center: [0, 0],
        preferCanvas: true,
        zoom: 2,
        zoomControl: false,
        worldCopyJump: true,
        layers: [
            L.tileLayer("https://{s}.google.com/vt/lyrs=m&hl=en&x={x}&y={y}&z={z}", {
                subdomains: ["mt0", "mt1", "mt2", "mt3"],
                type: "roadmap",
                maxZoom: 26,
                maxNativeZoom: 22
            })
        ]
    });


    customPolygonsLayer.addTo(map);
    map.addControl(drawControl);

    map.on("draw:created", (e) => {
        const polygon = e.layer;
        polygon.feature = e.layer.toGeoJSON();
        polygon.found = [];
        polygon.nbNeeded = 100;
        polygon.setStyle(customPolygonStyle());
        customPolygonsLayer.addLayer(polygon);
    });


    map.on("click", onClick);

    markerLayer.addTo(map);

    checkClusters();
});


</script>

<style>
@import "./assets/main.css";
#map {
	z-index: 0;
	height: 100vh;
}
.leaflet-container {
	background-color: #2c2c2c;
}
.overlay {
	position: absolute;
}
.select,
.selected,
.settings,
.export {
	padding: var(--space-2);
	border-radius: 5px;
	background: rgba(0, 0, 0, 0.7);
	box-shadow: 0 0 2px rgba(0, 0, 0, 0.4);
}
.selected {
	max-height: calc(100vh - 390px);
	overflow: auto;
}
.settings {
	max-width: 375px;
	max-height: calc(100vh - 180px);
	overflow: auto;
}
.export {
	min-width: 375px;
}
.line {
	line-height: 1.5rem;
}
</style>
