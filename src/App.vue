<template>
  <span>
    <div id="mapid"></div>
  </span>
</template>

<script>
import L from "leaflet";
import myGeoData from "../ph.json";

import parse from "csv-parse";
import axios from "axios";

export default {
  name: "App",
  data() {
    return {
      mymap: null,
      Confirmed: null,
      Deaths: null,
      Recovered: null,
      value: 0,
      fruits: 0,
      ticksLabels: ["Figs", "Lemon", "Pear", "Apple"]
    };
  },
  async mounted() {
    function getColor(d) {
      return d > 50000
        ? "#690000"
        : d > 17000
        ? "#BD0026"
        : d > 3000
        ? "#BD0026"
        : d > 1600
        ? "#E31A1C"
        : d > 800
        ? "#FC4E2A"
        : d > 400
        ? "#FD8D3C"
        : d > 200
        ? "#FEB24C"
        : d > 50
        ? "#FED976"
        : d > 0
        ? "#FFEDA0"
        : "";
    }
    function style(feature) {
      return {
        fillColor: getColor(feature.properties.Confirmed),
        weight: 0.5,
        opacity: 1,
        color: "white",
        dashArray: "3",
        fillOpacity: 0.7
      };
    }
    function onEachFeature(feature, layer) {
      if (feature.properties.Confirmed > 0) {
        layer.bindPopup(
          `<center> <strong> ${feature.properties.NAME_2} </strong> </center>
          Confirmed : <strong> ${feature.properties.Confirmed} </strong> <br>
          Deaths : <strong> ${feature.properties.Deaths} </strong> <br>
          Recovered : <strong> ${feature.properties.Recovered} </strong>`
        );
      }
    }

    var map = L.map("mapid", { zoomControl: false }).setView([12.8, 121.77], 6);
    L.tileLayer(
      "https://api.mapbox.com/styles/v1/{id}/tiles/{z}/{x}/{y}?access_token={accessToken}",
      {
        // attribution:   'Map data &copy; <a href="https://www.openstreetmap.org/">OpenStreetMap</a> contributors, <a href="https://creativecommons.org/licenses/by-sa/2.0/">CC-BY-SA</a>, Imagery © <a href="https://www.mapbox.com/">Mapbox</a>',
        maxZoom: 18,
        // id: "mapbox/streets-v11",
        id: "mapbox/dark-v9",
        tileSize: 512,
        zoomAnimation: false,
        zoomOffset: -1,
        preferCanvas: true,
        accessToken:
          "pk.eyJ1IjoicmV5bWFyYmMiLCJhIjoiY2syNDFzcjQxMDQxeTNobW4xdjE2OTEzbCJ9.29pNyPl-KBBLB8LCIiumGA"
      }
    ).addTo(map);
    L.control.scale({ position: "topright" }).addTo(map);

    const endDate = new Date();
    const lastDateString = `${endDate.getUTCMonth() +
      1}-${endDate.getUTCDate() - 1}-2020`;

    console.log(lastDateString);
    /*Legend specific*/
    var legend = L.control({ position: "bottomleft" });

    legend.onAdd = function() {
      var div = L.DomUtil.create("div", "legend");
      div.innerHTML += "<span>Confirmed Cases</span><br>";
      div.innerHTML +=
        '<i style="background: #FFEDA0"></i><span> > 1 </span><br>';
      div.innerHTML +=
        '<i style="background: #FED976"></i><span> > 50 </span><br>';
      div.innerHTML +=
        '<i style="background: #FEB24C"></i><span> > 200 </span><br>';
      div.innerHTML +=
        '<i style="background: #FD8D3C"></i><span> > 400 </span><br>';
      div.innerHTML +=
        '<i style="background: #FC4E2A"></i><span> > 800 </span><br>';
      div.innerHTML +=
        '<i style="background: #E31A1C"></i><span> > 1,600 </span><br>';
      div.innerHTML +=
        '<i style="background: #BD0026"></i><span> > 3,000 </span><br>';
      div.innerHTML +=
        '<i style="background: #800026"></i><span> > 17,000 </span><br>';
      div.innerHTML +=
        '<i style="background: #690000"></i><span> > 50,000+</span><br>';
      div.innerHTML +=
        ' <a href="https://geojson-maps.ash.ms/">Geo JSON</a> <br>';
      div.innerHTML +=
        ' <a href="https://github.com/CSSEGISandData/COVID-19/">CSSE DATA</a> <br>';
      div.innerHTML += `<i class="icon" 
          style="background-image: url(https://image.flaticon.com/icons/svg/25/25231.svg);
          background-repeat: no-repeat;"></i>   <a href="https://gyakusetsu.github.io/">Gyakusetsu</a> <br>`;

      return div;
    };

    legend.addTo(map);

    function onLocationError(e) {
      alert(e.message);
    }

    map.on("locationerror", onLocationError);

    // Create the parser
    const parser = parse({
      delimiter: ",",
      columns: true
    });

    let totalConfirmed = 0;
    let totalDeaths = 0;
    let totalRecovered = 0;
    let all_covid_data = [];
    let var_name = "";
    // Use the readable stream api
    parser.on("readable", function() {
      let covid_data;
      while ((covid_data = parser.read())) {
        covid_data[`${var_name}`] = parseInt(covid_data[`${var_name}`] || 0);
        all_covid_data.push(covid_data);

        // totalConfirmed += parseInt(covid_data[`Confirmed`]);
        // totalDeaths += parseInt(covid_data[`Deaths`]);
        // totalRecovered += parseInt(covid_data[`Recovered`]);
        if (covid_data.status.includes("Died")) {
          totalDeaths += 1;
        } else if (covid_data.status.includes("Recovered")) {
          totalConfirmed += 1;
        }

        totalConfirmed += 1;
      }
    });
    // Catch any error
    parser.on("error", function(err) {
      console.error(err.message);
    });
    // When we are done, test that the parsed output matched what expected
    parser.on("end", async function() {
      for await (const geo_data of myGeoData.features) {
        let currentCountry = null;

        // if (geo_data.properties.name == "United States") {
        //   currentCountry =
        //     all_covid_data.filter(c_data =>
        //       c_data[`Country_Region`].includes("US")
        //     ) || null;
        // } else {
        currentCountry =
          all_covid_data.filter(c_data =>
            c_data[`residence`]
              .replace(" City", "")
              .toLowerCase()
              .includes(
                geo_data.properties.NAME_2.replace(" City", "").toLowerCase()
              )
          ) || null;
        // }
        geo_data.properties[`Confirmed`] = 0;
        geo_data.properties[`Deaths`] = 0;
        geo_data.properties[`Recovered`] = 0;

        if (currentCountry.length > 0) {
          for (const iterator of currentCountry) {
            if (iterator.status.includes("Died")) {
              geo_data.properties[`Deaths`] += 1;
            } else if (iterator.status.includes("Recovered")) {
              geo_data.properties[`Recovered`] += 1;
            }

            geo_data.properties[`Confirmed`] += 1;
          }
        }
      }

      const geoJson = L.geoJson(myGeoData.features, {
        style: style,
        onEachFeature: onEachFeature
      });

      const debounce = (callback, delay = 250) => {
        let timeoutId;
        return (...args) => {
          clearTimeout(timeoutId);
          timeoutId = setTimeout(() => {
            timeoutId = null;
            callback(...args);
          }, delay);
        };
      };

      const addDebounce = debounce(() => {
        geoJson.addTo(map);
      }, 500);

      addDebounce();

      map.on("zoomstart", () => {
        map.removeLayer(geoJson);
      });

      map.on("zoomend", addDebounce);

      // map.locate({ setView: true, maxZoom: 4 });

      var totalLegend = L.control({ position: "topleft" });

      totalLegend.onAdd = function() {
        var div = L.DomUtil.create("div", "legend");
        div.innerHTML += `<center><strong><span>UTC: [${lastDateString}]</span></strong></center>`;
        div.innerHTML += `<center>Total Confirmed:  <br><strong><text style="color: red">${totalConfirmed}</text></strong></center>`;
        div.innerHTML += `<center>Total Deaths:<br><strong><text style="color: balck">${totalDeaths}</text></strong></center>`;
        div.innerHTML += `<center>Total Recovered: <br><strong><text style="color: green">${totalRecovered}</text></strong></center>`;

        return div;
      };

      totalLegend.addTo(map);
    });

    var_name = "Confirmed";
    this.Confirmed = await axios.get(
      `https://raw.githubusercontent.com/gigerbytes/ncov-ph-data/master/data/cases_ph.csv`
    );
    parser.write(this.Confirmed.data);
    // var_name = "Deaths";
    // this.Deaths = await axios.get(
    //   "https://raw.githubusercontent.com/CSSEGISandData/COVID-19/master/csse_covid_19_data/csse_covid_19_time_series/time_series_19-covid-Deaths.csv"
    // );
    // parser.write(this.Deaths.data);
    // this.Recovered = await axios.get(
    //   "https://raw.githubusercontent.com/CSSEGISandData/COVID-19/master/csse_covid_19_data/csse_covid_19_time_series/time_series_19-covid-Recovered.csv"
    // );
    // parser.write(this.Recovered.data, "Recovered");
    parser.end();
  }
};
</script>

<style>
/* #mapid {
  height: 100vh;
} */
</style>

<style lang="stylus">
@import '../node_modules/leaflet/dist/leaflet.css';

/* Setup */
html, body {
  padding: 0;
  margin: 0;
}

html, body {
  height: 100%;
  width: 100%;
}

#mapid {
  padding: 0;
  margin: 0;
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
}

.leaflet-container {
  background: #191a1a;
}
</style>