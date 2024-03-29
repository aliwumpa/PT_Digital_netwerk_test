<template>
  <div>
    <div class="leaflet-search-container">
      <input type="text" v-model="citySearch" placeholder="Search by city">
      <input type="text" v-model="hospitalNameSearch" placeholder="Search by hospital name">
      <button @click="search" :disabled="isFetching">Search</button>
    </div>
    <div class="leaflet-map-container">
      <div id="mapContainer"></div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from "vue";
// import axios  from "axios";
import "leaflet/dist/leaflet.css";
import L from "leaflet";

const API_key = 'AEYCtcSl7ZpwPxeYkHu4mg==8h3DXvljfPsZQdKq';
const proxy = 'https://thingproxy.freeboard.io/fetch/';
const hospitalUrl = 'https://dekontaminasi.com/api/id/covid19/hospitals';

const initialTiles = "http://{s}.tile.osm.org/{z}/{x}/{y}.png";
const initialIndonesiaCords = [-0.7893, 113.9213];
const initialZoom = 5;
const citySearch = ref<string>("");
const hospitalNameSearch = ref<string>("");
const refHospital = ref([]);
const isFetching = ref<boolean>(true);

let map;

onMounted(() => {
  // initiate map
  map = L.map("mapContainer").setView(initialIndonesiaCords, initialZoom);
  L.tileLayer(initialTiles, {
    attribution: '&copy; <a href="http://osm.org/copyright">OpenStreetMap</a> contributors'
  }).addTo(map);

  isFetching.value = true;

  fetchHospitalData();
});

const search = () => {
  const searchCity = citySearch.value.trim().toLowerCase();
  const searchHospital = hospitalNameSearch.value.trim().toLowerCase();

  if (searchCity || searchHospital) {
    const foundHospital = refHospital.value.find(hospital => {
      const hospitalCity = hospital.cityName;
      const hospitalName = hospital.name.toLowerCase();
      return hospitalCity.includes(searchCity) || hospitalName.includes(searchHospital);
    });

    if (foundHospital) {
      const latLng = L.latLng(foundHospital.lat, foundHospital.lng);
      const marker = L.marker([foundHospital.lat, foundHospital.lng]);
      marker.openPopup();
      map.setView(latLng, 13); 
    } else {
      alert("No matching hospital found.");
    }
  } else {
    alert("Enter city or hospital name to search.");
  }
};

const fetchHospitalData = () => {
  fetch(proxy + hospitalUrl)
  .then(response => {
      if (!response.ok) {
        throw new Error('Network response was not ok');
      }
      return response.json();
    })
    .then(async (hospitalData) => {
      for (const hospital of hospitalData) {
        // Pass parameter cityName data value to convert city to coordinates function
        const region = hospital.region;
        const regex = /KOTA\s([^,]+)/;
        let regionName = region.replace(regex, "$1");

        const coordinates = await convertCityToCoordinates(regionName);

        if(coordinates) {
          const { lat, lng } = coordinates;

          //store data to temp
          refHospital.value.push({
            name: hospital.name,
            address: hospital.address,
            cityName: regionName.toLowerCase(),
            lat,
            lng
          });

          const marker = L.marker([lat, lng]).addTo(map)
            .bindPopup(`<b>${hospital.name}</b><br>${hospital.address}`)
            .openPopup();
          marker.options.city = regionName.toLowerCase();
          marker.options.hospital = hospital.name.toLowerCase();
          marker.on('popupclose', () => {
            map.setView(initialIndonesiaCords, initialZoom); // Reset map view to default
          });
        }
      }

      isFetching.value = false;
    })
    .catch(error => {
      isFetching.value = false;
      console.error('There was a problem with the fetch operation:', error);
    });
};

const convertCityToCoordinates = async (city) => {
  try {
    const response = await fetch(`https://api.api-ninjas.com/v1/geocoding?city=${encodeURIComponent(city)}&country=indonesia`, {
      headers: {
        'X-Api-Key': API_key
      }
    });
    if (!response.ok) {
      throw new Error('Failed to geocode hospital address');
    }
    const data = await response.json();
    if(data.length !== 0) {
      const { latitude, longitude } = data[0];
      return { lat: latitude, lng: longitude };
    }
    return false;
  } catch (error) {
    console.error('Error geocoding hospital address:', error);
    return null;
  }
};

</script>

<style scoped>
.leaflet-search-container {
  padding: 30px;

  input {
    border-radius: 5px;
    padding: 10px;
    outline: none;
    border: 1px solid black;
  }

  input:nth-child(1), input:nth-child(2) {
    margin-right: 5px;
  }

  button {
    background-color: blueviolet;
    color: white;
    padding: 10px;
    outline: none;
    border-radius: 5px;
    margin-left: 5px;
    cursor: pointer;
  }

  button[disabled] {
    background-color: gray;
    color: white;
    cursor: not-allowed;
  }

  @media (max-width: 768px) {
    input:nth-child(1), input:nth-child(2) {
      margin-right: 0;
      margin-bottom: 10px;
      width: 100%;
    }

    button {
      margin-left: 0;
      margin-top: 10px;
    }
  }
}

.leaflet-map-container {
  width: 100%;
  height: auto;
}

/* #mapContainer styles should not be scoped */
#mapContainer {
  width: 100%;
  height: 100dvh;
}
</style>






