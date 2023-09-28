<template>
  <div>
    <button @click="getUserLocationAndDisplayMap">Show Map</button>
    <input type="text" v-model="searchQuery" placeholder="Search for a location" />
    <button @click="searchLocation">Search</button>
    <div id="map" style="width: 100%; height: 400px;"></div>

    <!-- List of added markers with clickable links and distance -->
    <div>
      <h3>Added Markers:</h3>
      <ul>
        <li v-for="(marker, index) in markers" :key="index">
          <a href="#" @click="centerMapOnMarker(marker)">
            {{ marker.title }}
          </a>
          <span v-if="userLocationMarker">
            Distance: {{ calculateDistance(userLocationMarker, marker).toFixed(2) }} miles
          </span>
          <span :style="{ color: isWithin3Miles(marker) ? 'green' : 'black' }">
            <span v-if="isWithin3Miles(marker)">
              (Within 3 miles of user)
            </span>
          </span>
        </li>
      </ul>
    </div>
  </div>
</template>

<script>
import { logger } from '../utils/Logger.js';

export default {
  data() {
    return {
      map: null,
      markers: [], // To store markers added to the map
      searchQuery: '', // User's search query
      searchService: null, // Google Places Autocomplete service
      isSettingMarker: false, // Flag to indicate if setting a marker is active
      pendingMarkerLocation: null, // Location selected for pending marker
      userLocationMarker: null, // Marker for the user's location
      infoWindows: [], // Store InfoWindow instances
    };
  },
  methods: {
    // Calculate distance between two markers
    calculateDistance(marker1, marker2) {
      const lat1 = marker1.getPosition().lat();
      const lng1 = marker1.getPosition().lng();
      const lat2 = marker2.getPosition().lat();
      const lng2 = marker2.getPosition().lng();

      const radlat1 = (Math.PI * lat1) / 180;
      const radlat2 = (Math.PI * lat2) / 180;
      const radlon1 = (Math.PI * lng1) / 180;
      const radlon2 = (Math.PI * lng2) / 180;
      const theta = lng1 - lng2;
      const radtheta = (Math.PI * theta) / 180;
      let dist =
        Math.sin(radlat1) * Math.sin(radlat2) +
        Math.cos(radlat1) * Math.cos(radlat2) * Math.cos(radtheta);
      dist = Math.acos(dist);
      dist = (dist * 180) / Math.PI;
      dist = dist * 60 * 1.1515; // Distance in miles (change to kilometers if needed)
      return dist;
    },

    getUserLocationAndDisplayMap() {
      if ('geolocation' in navigator) {
        navigator.geolocation.getCurrentPosition((position) => {
          const latitude = position.coords.latitude;
          const longitude = position.coords.longitude;

          this.map = new google.maps.Map(document.getElementById('map'), {
            center: { lat: latitude, lng: longitude },
            zoom: 12,
          });

          // Create a marker at the user's location
          this.userLocationMarker = new google.maps.Marker({
            position: { lat: latitude, lng: longitude },
            map: this.map,
            title: 'User',
          });

          // Add the userLocationMarker to the markers array
          this.markers.push(this.userLocationMarker);
        });
      } else {
        alert('Geolocation is not available in your browser');
      }
    },
    searchLocation() {
      if (this.searchQuery && this.map) {
        // Create a Places Autocomplete service
        if (!this.searchService) {
          this.searchService = new google.maps.places.AutocompleteService();
        }

        // Perform the autocomplete search
        this.searchService.getPlacePredictions(
          {
            input: this.searchQuery,
            componentRestrictions: { country: 'US' }, // Change to your desired country
          },
          (predictions) => {
            if (predictions && predictions.length > 0) {
              const place = predictions[0];

              // Get detailed information about the selected place
              const placeService = new google.maps.places.PlacesService(this.map);
              placeService.getDetails(
                { placeId: place.place_id },
                (result, status) => {
                  if (status === google.maps.places.PlacesServiceStatus.OK) {
                    // Center the map on the selected place
                    this.map.setCenter(result.geometry.location);
                  }
                }
              );
            }
          }
        );
      } else {
        alert('Map not initialized or search query is empty.');
      }
    },
    centerMapOnMarker(marker) {
      // Center the map on the clicked marker
      this.map.setCenter(marker.getPosition());
    },
    isWithin3Miles(marker) {
      if (this.userLocationMarker) {
        const distance = this.calculateDistance(this.userLocationMarker, marker);

        return distance <= 3; // Adjust the distance threshold as needed
      }
      return false;
    },
    createInfoWindow(marker) {
      const infowindow = new google.maps.InfoWindow({
        content: `<div>${marker.title}</div>`,
      });
      this.infoWindows.push(infowindow);
      return infowindow;
    },
    openInfoWindow(marker, infowindow) {
      this.infoWindows.forEach((iw) => iw.close());
      infowindow.open(this.map, marker);
    },
  },
  watch: {
    map(newValue) {
      if (newValue) {
        // Add a click event listener to the map for setting markers
        google.maps.event.addListener(newValue, 'click', (event) => {
          if (!this.isSettingMarker) {
            // Prompt the user to confirm setting a marker
            if (confirm('Do you want to set new Geo-Stache marker?')) {
              // Create a new marker at the clicked location
              const marker = new google.maps.Marker({
                position: event.latLng,
                map: this.map,
                title: 'Geo-Stache',
              });

              // Add the marker to the markers array
              this.markers.push(marker);
              logger.log(marker);

              // Create and open an InfoWindow with the marker's name
              const infowindow = this.createInfoWindow(marker);
              this.openInfoWindow(marker, infowindow);

              // Disable setting marker mode
              this.isSettingMarker = false;
            }
          }
        });
      }
    },
  },
  mounted() {
    if (typeof google !== 'undefined') {
      // Google Maps API is already loaded, call the function
      this.getUserLocationAndDisplayMap();
    } else {
      // Wait for the Google Maps API to be loaded
      const script = document.createElement('script');
      script.src = `https://maps.googleapis.com/maps/api/js?key=AIzaSyBifxFAXD3ecZoO52GpjV-STjO1LB1NnRg&libraries=places`;
      script.onload = this.getUserLocationAndDisplayMap;
      document.head.appendChild(script);
    }
  },
};
</script>

<style scoped lang="scss">
/* Your scoped CSS styles here */
</style>
