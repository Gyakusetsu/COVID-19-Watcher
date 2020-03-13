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
      div.innerHTML += "<span>Confirmed Cases</span><br>";
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
        '<i style="background: #800026"></i><span>1000 - 10000</span><br>';
      div.innerHTML +=
        '<i style="background: #690000"></i><span>10000+</span><br>';
      div.innerHTML +=
        ' <a href="https://github.com/CSSEGISandData/COVID-19/">CSSE DATA</a> <br>';
      div.innerHTML +=
        ' <a href="https://geojson-maps.ash.ms/">Geo JSON</a> <br>';

      return div;
    };

    legend.addTo(mymap);

    // let geoData = [];

    // let testdata = {};
    function getColor(d) {
      return d > 10000
        ? "#690000"
        : d > 1000
        ? "#BD0026"
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
          `<center> <strong> ${feature.properties.name} </strong> </center> 
          Confirmed : <strong> ${feature.properties.total_cases} </strong> <br> 
          Deaths : <strong> ${feature.properties.total_deaths} </strong> <br>
          Recovered : <strong> ${feature.properties.total_recovered} </strong>`
        );
      }
    }

    async function papaparseData(url, var_name, addToMap = false) {
      await Papa.parse(url, {
        download: true,
        header: true,
        complete: covid_19_data => {
          let covid_data = covid_19_data.data;
          const endDate = new Date();

          const lastDateString = `${endDate.getUTCMonth() +
            1}/${endDate.getUTCDate() - 1}/20`;

          console.log(lastDateString);

          covid_data.forEach(element => {
            element[`${var_name}`] = parseInt(
              element[`${lastDateString}`] || 0
            );
          });

          for (const geo_data of myGeoData.features) {
            let currentCountry = null;

            if (geo_data.properties.name == "United States") {
              currentCountry =
                covid_data.filter(c_data =>
                  c_data[`Country/Region`].includes("US")
                ) || null;
            } else {
              currentCountry =
                covid_data.filter(c_data =>
                  c_data[`Country/Region`].includes(geo_data.properties.name)
                ) || null;
            }

            if (currentCountry.length == 0) {
              geo_data.properties[`${var_name}`] = 0;
            } else {
              if (currentCountry.length > 1) {
                geo_data.properties[`${var_name}`] = 0;
                for (const iterator of currentCountry) {
                  geo_data.properties[`${var_name}`] += iterator[`${var_name}`];
                }
              } else {
                geo_data.properties[`${var_name}`] =
                  currentCountry[0][`${var_name}`];
              }
            }
          }

          if (addToMap) {
            console.log("finished");
            L.geoJson(myGeoData.features, {
              style: style,
              onEachFeature: onEachFeature
            }).addTo(mymap);
          }
        }
        // rest of config ...
      });
    }

    await papaparseData(
      "https://raw.githubusercontent.com/CSSEGISandData/COVID-19/master/csse_covid_19_data/csse_covid_19_time_series/time_series_19-covid-Confirmed.csv",
      "total_cases",
      false
    );
    await papaparseData(
      "https://raw.githubusercontent.com/CSSEGISandData/COVID-19/master/csse_covid_19_data/csse_covid_19_time_series/time_series_19-covid-Deaths.csv",
      "total_deaths",
      false
    );
    await papaparseData(
      "https://raw.githubusercontent.com/CSSEGISandData/COVID-19/master/csse_covid_19_data/csse_covid_19_time_series/time_series_19-covid-Recovered.csv",
      "total_recovered",
      true
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