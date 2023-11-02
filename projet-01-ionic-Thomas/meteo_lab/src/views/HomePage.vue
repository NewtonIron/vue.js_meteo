<template>
  <!-- Définition de la structure de la page -->
  <ion-page class="background">
    <!-- En-tête de la page -->
    <ion-header>
      <!-- Barre d'outils avec titre dynamique -->
      <ion-toolbar color="primary" class="ion-text-center">
        <ion-title>Meteo:  {{ cityName }}</ion-title>
      </ion-toolbar>
      <!-- Liste déroulante pour sélectionner la position -->
      <ion-list>
        <ion-item class="ion-align-items-start">
          <!-- Sélecteur de position avec gestion de l'événement "ionChange" -->
          <ion-select label="Sélectionnez la position" labelPlacement="floating" fill="solid" placeholder="Position" v-model="selectedLocation" @ionChange="onLocationChange" class="ion-align-items-start" >
            <!-- Options de sélection avec valeurs et labels -->
            <ion-select-option value="45.5088,-73.5878">Montréal</ion-select-option>
            <ion-select-option value="45.57,-73.692">Laval</ion-select-option>
            <ion-select-option value="46.8123,-71.2145">Quebec</ion-select-option>
            <ion-select-option value="currentLocation">Ma position</ion-select-option>
          </ion-select>
        </ion-item>
      </ion-list>
    </ion-header>
    <!-- Contenu de la page -->
    <ion-content>
      <div class="container">
        <div class="content">
          <!-- Titre de la page avec nom de la ville dynamique -->
          <h1>Météo pour {{ cityName }}</h1>
          <!-- Affichage des informations météorologiques si elles sont disponibles -->
          <div v-if="weatherData" class="weather-info">
            <!-- Image de la météo -->
            <img class="icone" :src="weatherImage" alt="Image de la météo" />
            <ion-text>
              <!-- Affichage de la température, de l'heure, de l'humidité et de la description de la météo -->
              <p>Température : {{ weatherData.current.temp }}°C</p>
              <p>heure : {{ formattedTime }}h</p>
              <p>Humidité : {{ weatherData.current.humidity }}%</p>
              <p>Description de la météo : {{ weatherDescription }}</p>
            </ion-text>
          </div>
          <!-- Affichage d'un message d'erreur en cas de problème -->
          <div v-else-if="error" class="error-message">
            <p>{{ error }}</p>
          </div>
        </div>
      </div>
    </ion-content>
    <!-- Pied de page avec un titre dynamique -->
    <ion-footer>
      <ion-toolbar color="primary" class="ion-text-center">
        <ion-title>Meteo:  {{ cityName }}</ion-title>
      </ion-toolbar>
    </ion-footer>
  </ion-page>
</template>

<script lang="ts">
// Imports des bibliothèques et composants nécessaires
import { ref, onMounted, computed } from 'vue';
import axios from 'axios';
import cloudyImage from '@/assets/03d.svg';
import couvertImage from '@/assets/04d.svg';
import pluiImage from '@/assets/09d.svg';
import cielImage from '@/assets/01d.svg';
import neigeImage from '@/assets/13d.svg';
import { Geolocation } from '@capacitor/geolocation';

import {
  IonContent,
  IonHeader,
  IonPage,
  IonTitle,
  IonToolbar,
  IonSelect,
  IonSelectOption,
  IonItem,
  IonList,
  IonText,
  IonFooter
} from '@ionic/vue';
import { defineComponent } from 'vue';

// Interface pour représenter les données météorologiques
interface Weather {
  current: {
    temp: number;
    humidity: number;
    dt: number;
    weather: { description: string }[];
    timezone: { description: string }[];
  };
}

// Classe pour gérer la géolocalisation
class GeoLoc {
  latitude: number;
  longitude: number;

  constructor() {
    this.latitude = 0;
    this.longitude = 0;
  }

  // Méthode pour obtenir la position actuelle
  async getPosition() {
    try {
      const position = await Geolocation.getCurrentPosition();
      this.latitude = position.coords.latitude;
      this.longitude = position.coords.longitude;
    } catch (error) {
      console.error('Erreur lors de la récupération de la position:', error);
      throw error; // Répétez l'erreur pour la capturer dans la gestion Axios
    }
  }
}

export default defineComponent({
  components: {
    IonContent,
    IonHeader,
    IonPage,
    IonTitle,
    IonToolbar,
    IonSelect,
    IonSelectOption,
    IonItem,
    IonList,
    IonText,
    IonFooter
  },
  setup() {
    // Références aux données et états
    const selectedLocation = ref('45.5088,-73.5878'); // Coordonnées de Montréal par défaut
    const weatherData = ref<Weather | null>(null);
    const error = ref<string | null>(null);
    const geoLoc = new GeoLoc(); // Crée une instance de GeoLoc
    const formattedTime = ref<string>('');

    // Fonction pour récupérer les données météorologiques
    const fetchWeather = async () => {
      try {
        const [latitude, longitude] = selectedLocation.value.split(',');
        const response = await axios.get('https://api.openweathermap.org/data/3.0/onecall', {
          params: {
            lat: latitude,
            lon: longitude,
            exclude: 'hourly,daily',
            appid: 'eb2e7652fb85856210248f050dc2c102',
            units: 'metric',
            lang: 'fr',
          },
        });

        const weatherResponse: Weather = response.data;

        if (weatherResponse && weatherResponse.current) {
          // Stockage des données météorologiques
          weatherData.value = weatherResponse;
          const currentTimeUnix = weatherData.value.current.dt;
          const currentTime = new Date(currentTimeUnix * 1000); // Convertissez en millisecondes

          const hours = currentTime.getHours();
          const minutes = currentTime.getMinutes();
          const seconds = currentTime.getSeconds();
          formattedTime.value = `${hours}:${minutes}:${seconds}`;
        } else {
          throw new Error('Données météo non valides');
        }
      } catch (err) {
        console.error('Erreur lors de la récupération des données météo:', err);
        error.value = 'Une erreur s\'est produite lors de la récupération des données météo.';
        throw err; // Répétez l'erreur pour la capturer dans la gestion Axios
      }
    };

    // Fonction appelée lorsqu'une nouvelle position est sélectionnée
    const onLocationChange = async () => {
      if (selectedLocation.value === 'currentLocation') {
        try {
          // Obtention de la position actuelle
          await geoLoc.getPosition();
          selectedLocation.value = `${geoLoc.latitude},${geoLoc.longitude}`;
        } catch (err) {
          console.error('Erreur lors de la récupération de la position:', err);
          error.value = 'Impossible d\'obtenir la position actuelle.';
          return;
        }
      }
      
      // Actualisation des données météorologiques
      fetchWeather();
    };

    // Appeler fetchWeather lorsque le composant est monté
    onMounted(fetchWeather);

    // Données des villes pour afficher le nom de la ville sélectionnée
    const citiesData: { [key: string]: string } = {
      '45.5088,-73.5878': 'Montréal',
      '45.57,-73.692': 'Laval',
      '46.8123,-71.2145': 'Quebec'
    };

    // Calculer le nom de la ville en fonction de la position sélectionnée
    const cityName = computed(() => {
      return citiesData[selectedLocation.value] || 'votre position';
    });

    // Calculer la description de la météo
    const weatherDescription = computed(() => {
      if (weatherData.value && weatherData.value.current.weather && weatherData.value.current.weather.length) {
        return weatherData.value.current.weather[0].description;
      }
      return '';
    });

    // Calculer l'image en fonction de la description de la météo
    const weatherImage = computed(() => {
      if (weatherDescription.value.includes('nuageux')) {
        return cloudyImage;
      } else if (weatherDescription.value.includes('couvert')){
        return couvertImage;
      } else if (weatherDescription.value.includes('pluie')){
        return pluiImage;
      } else if (weatherDescription.value.includes('ciel dégagé')){
        return cielImage;
      }else if (weatherDescription.value.includes('neige')){
        return neigeImage;
      }
      return '';
    });

    // Retourner les données et fonctions pour l'utilisation dans le template
    return {
      selectedLocation,
      weatherData,
      error,
      cityName,
      weatherDescription,
      weatherImage,
      onLocationChange,
      geoLoc,
      formattedTime,
    };
  },
});
</script>

<style scoped>
  /* Style de l'arrière-plan de la page */
  ion-content {
    --background: url('../assets/background-morning.jpeg');
  }
  .icone {
  width: 50vw; /* Réglez la largeur en pourcentage de la largeur de la fenêtre */
  height: 50vh;
  max-height: 100%; /* Assurez-vous que la hauteur s'ajuste proportionnellement */
  max-width: 100%; /* Empêche l'image de dépasser la largeur du conteneur */
}
.container {
    text-align: center;
  }
  
  .content {
    padding: 20px;
    border-radius: 10px;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
  }
  
  .weather-info {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
  }
  
  .weather-info p {
    margin: 5px 0; /* Ajoute une petite marge en haut et en bas pour espacer les éléments <p> */
  }


</style>
