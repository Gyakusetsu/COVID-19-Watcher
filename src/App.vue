<template>
  <div id="mapid"></div>
</template>

<script>
import L from "leaflet";
import Papa from "papaparse";
import myGeoData from "../custom.geo.json";

export default {
  name: "App",
  async mounted() {
    var mymap = L.map("mapid").setView([51.505, -0.09], 2);
    L.tileLayer(
      "https://api.mapbox.com/styles/v1/{id}/tiles/{z}/{x}/{y}?access_token={accessToken}",
      {
        // attribution:   'Map data &copy; <a href="https://www.openstreetmap.org/">OpenStreetMap</a> contributors, <a href="https://creativecommons.org/licenses/by-sa/2.0/">CC-BY-SA</a>, Imagery © <a href="https://www.mapbox.com/">Mapbox</a>',
        maxZoom: 18,
        // id: "mapbox/streets-v11",
        id: "mapbox/dark-v9",
        tileSize: 512,
        zoomOffset: -1,
        accessToken:
          "pk.eyJ1IjoicmV5bWFyYmMiLCJhIjoiY2syNDFzcjQxMDQxeTNobW4xdjE2OTEzbCJ9.29pNyPl-KBBLB8LCIiumGA"
      }
    ).addTo(mymap);
    L.control.scale().addTo(mymap);

    /*Legend specific*/
    var legend = L.control({ position: "bottomleft" });

    legend.onAdd = function() {
      var div = L.DomUtil.create("div", "legend");
      div.innerHTML += "<h4>Tegnforklaring</h4>";
      div.innerHTML +=
        '<i style="background: #FFEDA0"></i><span>1 - 10</span><br>';
      div.innerHTML +=
        '<i style="background: #FED976"></i><span>10 - 20</span><br>';
      div.innerHTML +=
        '<i style="background: #FEB24C"></i><span>20 - 50</span><br>';
      div.innerHTML +=
        '<i style="background: #FD8D3C"></i><span>50 - 100</span><br>';
      div.innerHTML +=
        '<i style="background: #FC4E2A"></i><span>100 - 200</span><br>';
      div.innerHTML +=
        '<i style="background: #E31A1C"></i><span>200 - 500</span><br>';
      div.innerHTML +=
        '<i style="background: #BD0026"></i><span>500 - 1000</span><br>';
      div.innerHTML +=
        '<i style="background: #800026"></i><span>1000+</span><br>';
  
      return div;
    };

    legend.addTo(mymap);

    // let geoData = [];

    // let testdata = {};
    function getColor(d) {
      return d > 1000
        ? "#800026"
        : d > 500
        ? "#BD0026"
        : d > 200
        ? "#E31A1C"
        : d > 100
        ? "#FC4E2A"
        : d > 50
        ? "#FD8D3C"
        : d > 20
        ? "#FEB24C"
        : d > 10
        ? "#FED976"
        : d > 1
        ? "#FFEDA0"
        : "#FFFFFF";
    }
    function style(feature) {
      return {
        fillColor: getColor(feature.properties.total_cases),
        weight: 2,
        opacity: 1,
        color: "white",
        dashArray: "3",
        fillOpacity: 0.7
      };
    }
    // function binPopUp(e) {
    //   map.fitBounds(e.target.getBounds());
    // }
    function onEachFeature(feature, layer) {
      // layer.on({
      //   mouseover: highlightFeature,
      //   mouseout: resetHighlight,
      //   click: zoomToFeature
      // });
      if (feature.properties.total_cases > 0) {
        layer.bindPopup(
          `${feature.properties.name} : <strong> ${feature.properties.total_cases} </strong>`
        );
      }
    }

    Papa.parse(
      "https://raw.githubusercontent.com/CSSEGISandData/COVID-19/master/csse_covid_19_data/csse_covid_19_time_series/time_series_19-covid-Confirmed.csv",
      {
        download: true,
        header: true,
        complete: covid_19_data => {
          let covid_data = covid_19_data.data;
          covid_data.forEach(element => {
            const initialDate = new Date("1/22/2020");
            let currentDate = initialDate;
            const endDate = new Date();

            const currentDateString = `${endDate.getMonth() +
              1}/${endDate.getDate() - 1}/20`;

            console.log(currentDateString);

            element.total_cases = parseInt(
              element[`${currentDateString}`] || 0
            );

            while (currentDate <= endDate) {
              const currentDateString = `${currentDate.getMonth() +
                1}/${currentDate.getDate()}/20`;
              currentDate.setDate(currentDate.getDate() + 1);

              // element.total_cases += parseInt(
              //   element[`${currentDateString}`] || 0
              // );
              delete element[`${currentDateString}`];
            }
          });

          let customGeo = JSON.parse(JSON.stringify(myGeoData));

          for (const geo_data of customGeo.features) {
            const currentCountry =
              covid_data.find(
                c_data => c_data[`Country/Region`] == geo_data.properties.name
              ) || null;

            if (currentCountry == null) {
              geo_data.properties.total_cases = 0;
              continue;
            }
            geo_data.properties.total_cases = currentCountry.total_cases;
          }

          L.geoJson(customGeo.features, {
            style: style,
            onEachFeature: onEachFeature
          }).addTo(mymap);
        }
        // rest of config ...
      }
    );
  }
};
</script>

<style>
#mapid {
  height: 100vh;
}
</style>

<style lang="stylus">
@import '../node_modules/leaflet/dist/leaflet.css';

/* Setup */
html, body {
  padding: 0;
  margin: 0;
}

html, body, #mapid {
  height: 100%;
  width: 100%;
}

/* Legend specific */
.legend {
  padding: 6px 8px;
  font: 14px Arial, Helvetica, sans-serif;
  background: white;
  background: rgba(255, 255, 255, 0.8);
  /* box-shadow: 0 0 15px rgba(0, 0, 0, 0.2); */
  /* border-radius: 5px; */
  line-height: 24px;
  color: #555;
}

.legend h4 {
  text-align: center;
  font-size: 16px;
  margin: 2px 12px 8px;
  color: #777;
}

.legend span {
  position: relative;
  bottom: 3px;
}

.legend i {
  width: 18px;
  height: 18px;
  float: left;
  margin: 0 8px 0 0;
  opacity: 0.7;
}

.legend i.icon {
  background-size: 18px;
  background-color: rgba(255, 255, 255, 1);
}
</style>