<template>
  <main class="gabty">
    <div class="sidebar">
      <img src="../assets/logo.png" alt="Logo" class="logo" />

      <ul class="workouts">
        <b-form
          @submit.prevent="newWorkout"
          class="form"
          :class="{ hidden: hideForm }"
        >
          <div class="form__row">
            <label class="form__label">Type</label>
            <select
              @change="toggleElevate = !toggleElevate"
              class="form__input form__input--type"
              ref="gType"
            >
              <option value="running">Running</option>
              <option value="cycling">Cycling</option>
            </select>
          </div>

          <div class="form__row">
            <label class="form__label">Distance</label>
            <input
              class="form__input form__input--distance"
              placeholder="km"
              ref="gDistance"
            />
          </div>

          <div class="form__row">
            <label class="form__label">Duration</label>
            <input
              class="form__input form__input--duration"
              placeholder="min"
              ref="gDuration"
            />
          </div>

          <div
            class="form__row"
            :class="toggleElevate ? 'form__row--hidden' : ''"
          >
            <label class="form__label">Cadence</label>
            <input
              class="form__input form__input--cadence"
              placeholder="step/min"
              ref="gCadence"
            />
          </div>
          <div
            class="form__row"
            :class="toggleElevate ? '' : 'form__row--hidden'"
          >
            <label class="form__label">Elev Gain</label>
            <input
              class="form__input form__input--elevation"
              placeholder="meters"
              ref="gElevation"
            />
          </div>
          <button class="form__btn">OK</button>
        </b-form>

        <li
          v-for="workout in workouts"
          :key="workout"
          class="workout"
          :class="`workout--${workout.type}`"
          :data-id="workout.id"
          ref="workoutContainer"
          @click="moveToPopup"
        >
          <h2 class="workout__title">{{ workout.description }}</h2>
          <div class="workout__details">
            <span class="workout__icon">{{
              workout.type === "running" ? "🏃‍♂️" : "🚴‍♀️"
            }}</span>
            <span class="workout__value">{{ workout.distance }}</span>
            <span class="workout__unit">km</span>
          </div>
          <div class="workout__details">
            <span class="workout__icon">⏱</span>
            <span class="workout__value">{{ workout.duration }}</span>
            <span class="workout__unit">min</span>
          </div>

          <div v-if="workout.type === 'running'" class="workout__details">
            <span class="workout__icon">⚡️</span>
            <span class="workout__value">{{ workout.pace.toFixed(1) }}</span>
            <span class="workout__unit">min/km</span>
          </div>
          <div v-if="workout.type === 'running'" class="workout__details">
            <span class="workout__icon">🦶🏼</span>
            <span class="workout__value">{{ workout.cadence }}</span>
            <span class="workout__unit">spm</span>
          </div>

          <div v-if="workout.type === 'cycling'" class="workout__details">
            <span class="workout__icon">⚡️</span>
            <span class="workout__value">{{ workout.speed.toFixed(1) }}</span>
            <span class="workout__unit">km/h</span>
          </div>
          <div v-if="workout.type === 'cycling'" class="workout__details">
            <span class="workout__icon">⛰</span>
            <span class="workout__value">{{ workout.elevationGain }}</span>
            <span class="workout__unit">m</span>
          </div>
        </li>
      </ul>
    </div>

    <div ref="map" id="map"></div>
  </main>
</template>

<script>
import { Running, Cycling } from "../utils/WorkoutClass";

export default {
  name: "Gabty",
  data() {
    return {
      hideForm: true,
      workouts: [],
      toggleElevate: false,
      mapEvent: {},
      map: {},
    };
  },
  mounted() {
    this.getPosition();

    try {
      this.workouts = JSON.parse(localStorage.getItem("workouts")) || [];
    } catch (err) {
      return;
    }
  },
  methods: {
    getPosition() {
      if (navigator.geolocation)
        navigator.geolocation.getCurrentPosition(
          this.handleUserCurrentLocation,
          function () {
            alert("Could not get your position");
          }
        );
    },

    handleUserCurrentLocation(position) {
      this.loadMap(position);
    },

    loadMap(position) {
      const { latitude, longitude } = position.coords;

      const coords = [latitude, longitude];

      this.map = L.map("map", { zoomControl: false }).setView(coords, 13);

      L.tileLayer(
        "https://{s}.tile.openstreetmap.fr/hot/{z}/{x}/{y}.png"
      ).addTo(this.map);

      this.map.on("click", this.showForm);

      this.workouts.forEach(
        function (work) {
          this.renderWorkoutMarker(work);
        }.bind(this)
      );
    },

    showForm(mapEvent) {
      this.hideForm = false;
      this.mapEvent = mapEvent;
    },

    newWorkout() {
      const validInputs = (...inputs) =>
        inputs.every((inp) => Number.isFinite(inp));
      const allPositive = (...inputs) => inputs.every((inp) => inp > 0);

      const type = this.$refs.gType.value;
      const distance = parseInt(this.$refs.gDistance.value);
      const duration = parseInt(this.$refs.gDuration.value);
      const { lat, lng } = this.mapEvent.latlng;
      let workout;

      if (type === "running") {
        const cadence = parseInt(this.$refs.gCadence.value);

        if (
          !validInputs(distance, duration, cadence) ||
          !allPositive(distance, duration, cadence)
        )
          return alert("Inputs have to be positive numbers!");

        workout = new Running([lat, lng], distance, duration, cadence);
      }

      // If workout cycling, create cycling object
      if (type === "cycling") {
        const elevation = parseInt(this.$refs.gElevation.value);

        if (
          !validInputs(distance, duration, elevation) ||
          !allPositive(distance, duration)
        )
          return alert("Inputs have to be positive numbers!");

        workout = new Cycling([lat, lng], distance, duration, elevation);
      }

      // Add new object to workout array
      this.workouts.unshift(workout);

      this.renderWorkoutMarker(workout);

      this.hideForm = true;

      this.setLocalStorage();
    },

    renderWorkoutMarker(workout) {
      L.marker(workout.coords)
        .addTo(this.map)
        .bindPopup(
          L.popup({
            maxWidth: 250,
            minWidth: 100,
            autoClose: false,
            closeOnClick: false,
            className: `${workout.type}-popup`,
          })
        )
        .setPopupContent(
          `${workout.type === "running" ? "🏃‍♂️" : "🚴‍♀️"} ${workout.description}`
        )
        .openPopup();
    },

    moveToPopup() {
      if (!this.map) return;

      const workoutId = this.$refs.workoutContainer[0].dataset.id;

      const workout = this.workouts.find((work) => work.id === workoutId);

      this.map.setView(workout.coords, 13, {
        animate: true,
        pan: {
          duration: 1,
        },
      });
    },

    setLocalStorage() {
      localStorage.setItem("workouts", JSON.stringify(this.workouts));
    },
  },
};
</script>

<style scoped>
.gabty {
  position: relative;
  font-family: "Manrope", sans-serif;
  color: var(--color-light--2);
  font-weight: 400;
  line-height: 1.6;
  height: 100vh;
  overscroll-behavior-y: none;

  background-color: #fff;
  padding: 2.5rem;

  display: flex;
}

/* MAP */
#map {
  flex: 1;
  height: 100%;
  background-color: var(--color-light--1);
  z-index: 0;
}

@media (max-width: 1234px) {
  .gabty {
    flex-direction: column;
  }
}

/* GENERAL */
a:link,
a:visited {
  color: var(--color-brand--1);
}

/* SIDEBAR */
.sidebar {
  position: absolute;
  bottom: 40px;
  top: 40px;
  left: 40px;
  width: 590px;
  background-color: var(--color-dark--1);
  padding: 3rem 5rem 4rem 5rem;
  display: flex;
  flex-direction: column;
  border-radius: 10px;
  z-index: 10;
}

@media (max-width: 1234px) {
  .sidebar {
    position: static;
    height: 320px;
    width: auto;
    border-bottom-left-radius: 0%;
    border-bottom-right-radius: 0;
  }
}

.logo {
  height: 5.2rem;
  align-self: center;
  margin-bottom: 4rem;
}

.workouts {
  list-style: none;
  height: 77vh;
  overflow-y: scroll;
  overflow-x: hidden;
}

.workouts::-webkit-scrollbar {
  width: 0;
}

.workout {
  background-color: var(--color-dark--2);
  border-radius: 5px;
  padding: 1.5rem 2.25rem;
  margin-bottom: 1.75rem;
  cursor: pointer;

  display: grid;
  grid-template-columns: 1fr 1fr 1fr 1fr;
  gap: 0.75rem 1.5rem;
}
.workout--running {
  border-left: 5px solid var(--color-brand--2);
}
.workout--cycling {
  border-left: 5px solid var(--color-brand--1);
}

.workout__title {
  font-size: 1.7rem;
  font-weight: 600;
  grid-column: 1 / -1;
}

.workout__details {
  display: flex;
  align-items: baseline;
}

.workout__icon {
  font-size: 1.8rem;
  margin-right: 0.2rem;
  height: 0.28rem;
}

.workout__value {
  font-size: 1.5rem;
  margin-right: 0.5rem;
}

.workout__unit {
  font-size: 1.1rem;
  color: var(--color-light--1);
  text-transform: uppercase;
  font-weight: 800;
}

.form {
  background-color: var(--color-dark--2);
  border-radius: 5px;
  padding: 1.5rem 2.75rem;
  margin-bottom: 1.75rem;

  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 0.5rem 2.5rem;

  /* Match height and activity boxes */
  height: 9.25rem;
  transition: all 0.5s, transform 1ms;
}

.form.hidden {
  transform: translateY(-30rem);
  height: 0;
  padding: 0 2.25rem;
  margin-bottom: 0;
  opacity: 0;
}

.form__row {
  display: flex;
  align-items: center;
}

.form__row--hidden {
  display: none;
}

.form__label {
  flex: 0 0 50%;
  font-size: 1.5rem;
  font-weight: 600;
}

.form__input {
  width: 100%;
  padding: 0.3rem 1.1rem;
  font-family: inherit;
  font-size: 1.4rem;
  border: none;
  border-radius: 3px;
  background-color: var(--color-light--3);
  transition: all 0.2s;
}

.form__input:focus {
  outline: none;
  background-color: #fff;
}

.form__btn {
  display: none;
}

.copyright {
  margin-top: auto;
  font-size: 1.3rem;
  text-align: center;
  color: var(--color-light--1);
}

.twitter-link:link,
.twitter-link:visited {
  color: var(--color-light--1);
  transition: all 0.2s;
}

.twitter-link:hover,
.twitter-link:active {
  color: var(--color-light--2);
}

/* Popup width is defined in JS using options */
.leaflet-popup .leaflet-popup-content-wrapper {
  background-color: var(--color-dark--1);
  color: var(--color-light--2);
  border-radius: 5px;
  padding-right: 0.6rem;
}

.leaflet-popup .leaflet-popup-content {
  font-size: 1.5rem;
}

.leaflet-popup .leaflet-popup-tip {
  background-color: var(--color-dark--1);
}

.running-popup .leaflet-popup-content-wrapper {
  border-left: 5px solid var(--color-brand--2);
}
.cycling-popup .leaflet-popup-content-wrapper {
  border-left: 5px solid var(--color-brand--1);
}

.leaflet-zoom-box {
  display: none;
}
</style>
