<script>
import ewgData from '../../src/static/ewg.json';
import ewgAircrafts from '../../src/static/ewgAircrafts.json';
import type { Trip, Route } from '../../src/types/types';
import { TripService } from '../../src/services/trip-service';
import { FilterUtils } from '../../src/utils/filters';

let selectedCallsign = '';
let availableCallsigns = [];
let hoursLimit = 0;
let selectedAircraft = '';

$: aircrafts = selectedCallsign ? ewgAircrafts[selectedCallsign] : ewgAircrafts['EWG'];

$: allRoutes = ewgData.routes.sort((a, b) => FilterUtils.sortAlphabetically(a.icao, b.icao));

$: routes = getFilteredRoute(selectedCallsign, hoursLimit, selectedAircraft);

function getFilteredRoute(selectedCallsign, hoursLimit, selectedAircraft) {
    const routeFilteredByCallsing = selectedCallsign
        //@ts-ignore
        ? FilterUtils.filterDataByCallsign(allRoutes, selectedCallsign)
        : allRoutes.value;

    const routeFilteredByHours = hoursLimit !== 0
        ? FilterUtils.filterDataByHours(routeFilteredByCallsing, hoursLimit)
        : routeFilteredByCallsing;

    const routesFilteredByAircraft = selectedAircraft
        ? FilterUtils.filterDataByAircraft(routeFilteredByHours, selectedAircraft)
        : routeFilteredByHours;

    return routesFilteredByAircraft;
}
</script>

<div class="popup-body ewg-theme">
  <div class="form-container">
    <div>
      <span> Airline: </span>
      <select bind:value={selectedCallsign}>
        {#each availableCallsigns as callsign (callsign)}
            <option>
              {{ callsign }}
            </option>
        {/each}
      </select>
    </div>
    <div>
      <span> Aircraft: </span>
      <select v-model="selectedAircraft">
        <option
          v-if="aircrafts?.length"
          :value="null">
        </option>
        <option v-for="aircraft in aircrafts" :key="aircraft" :value="aircraft">
          {{ aircraft }}
        </option>
      </select>
    </div>
    <div>
      <span> Departure: </span>
      <select v-model="selectedDeparture">
        <option
          v-if="routes?.length"
          :value="null">
        </option>
        <option v-for="airport in departureRoutes" :key="airport.id" :value="airport">
          {{ airport.icao }} ( {{ airport.name }} )
        </option>
      </select>
      <span style="display: flex; align-items: center; margin-top: 5px;">
                    <label for="returnToBase">
                        Start and return at base:
                    </label>
                    <input type="checkbox" id="return-to-base" name="returnToBase" :checked="isStartAndReturnAtBase" @click="isStartAndReturnAtBase = !isStartAndReturnAtBase" />
                </span>
    </div>
    <div v-if="selectedDeparture && !isStartAndReturnAtBase">
      <span> Destination: </span>
      <select
        v-model="selectedDestination">
        <option
          v-if="routes?.length"
          :value="null">
        </option>
        <option v-for="airport in routes" :key="airport.id" :value="airport">
          {{ airport.icao }} ( {{ airport.name }} )
        </option>
      </select>
    </div>
    <div>
      <span> Leg number: </span>
      <select
        v-model="legNumber">
        <option v-for="leg in legCountOptions" :key="leg" :value="leg">
          {{ leg }}
        </option>
      </select>
    </div>
    <div>
      <span> Hour Limit per leg: </span>
      <select
        v-model="hoursLimit">
        <option v-for="hour in hoursLimitOptions" :key="hour" :value="hour">
          {{ hour > 0 ? hour : 'No limit' }}
        </option>
      </select>
    </div>
  </div>
  <button
    class="search-button"
    :disabled="isGeneratingRoute"
    @click="calculateLegs">
    Search
  </button>

  <span v-if="isGeneratingRoute">
            Generating Route...
        </span>
  <div v-if="foundTrip?.legs?.length">
    <h2>Found Trip</h2>
    <p>
      Aircraft:
      <span class="generated-aircraft-type">
                    {{ generatedAircraftType }}
                </span>
    </p>
    <h2> Route: </h2>
    <table>
      <thead>
      <tr>
        <th>#</th>
        <th>Origin</th>
        <th>Destination</th>
        <th>Duration</th>
      </tr>
      </thead>
      <tbody>
      <tr v-for="(leg, index) in foundTrip?.legs">
        <td>{{ index + 1 }}</td>
        <td>{{ leg.start }}</td>
        <td>{{ leg.end }}</td>
        <td>
          {{ allRoutes?.find((route) => route.icao === leg.start)?.destinations?.find((destination) => destination.icao === leg.end)?.time }}
        </td>
      </tr>
      <tr>
        <td v-if="!totalTripTime?.includes('NaN') && !totalTurnaroundTime?.includes('NaN')" colspan="4">
          Total trip time: <span style="font-weight: bold"> {{ totalTripTime }} </span> (includes {{ totalTurnaroundTime }} of turnaroud)
        </td>
      </tr>
      </tbody>
    </table>
  </div>
  <div class="error-message" v-else-if="generatedAircraftType || isError">
            <span v-if="routes?.length === 0">
                No possible trip found for selected parameters. Try changing parameters.
            </span>
    <span v-else-if="generatedAircraftType">
                No trip found for generated aircraft:
      {{ generatedAircraftType }}  and {{generatedDeparture}} departure airport,
                it might take a few attempts. Try changing parameters or search again.
            </span>
    <span v-else>
                No trip found for selected parameters. Try changing parameters or search again.
            </span>
  </div>
</div>






<style>
    .popup-body {
        width: 400px;
        padding: 30px;
        font-family: Arial, sans-serif;
        box-sizing: border-box;
        font-size: 12px;
        margin: 20px;
    }
    @media (min-width: 600px) {
        .popup-body {
            border-radius: 5px;
        }

        .ewg-theme {
            border: 1px solid #666666;
        }
    }

    .form-container {
        display: flex;
        flex-direction: column;
        gap: 10px;
        margin-bottom: 20px;
    }

    .form-container > div > span {
        font-weight: bold;
        margin-bottom: 5px;
        font-size: 12px;
    }

    .form-container > select {
        padding: 8px;
        border: 1px solid #ccc;
        border-radius: 4px;
    }

    .form-container > div {
        display: flex;
        flex-direction: column;
    }

    button {
        background-color: #007bff;
        color: white;
        padding: 10px 20px;
        border: none;
        border-radius: 4px;
        cursor: pointer;
        transition: background-color 0.3s ease;
    }

    button:hover {
        background-color: #0056b3;
    }

    button:active {
        background-color: #00428a;
    }

    button:disabled {
        opacity: 0.6;
        cursor: default;
    }

    table {
        width: 100%;
        border-collapse: collapse;
    }

    th, td {
        border: 1px solid #ddd;
        padding: 8px;
    }

    th {
        text-align: left;
        background-color: #f2f2f2;
    }

    h2 {
        font-size: 1.2em;
        margin-bottom: 10px;
    }

    .generated-aircraft-type {
        font-weight: bold;
    }

    .error-message {
        color: #d9534f;
        background-color: #ffe0e0;
        border: 1px solid #d43f3a;
        padding: 10px;
        margin: 15px 0;
        max-width: 380px;
        word-wrap: break-word;
    }

    .ewg-theme {
        background-color: #F9F9FC;
        color: #000;
    }

    .ewg-theme table th {
        background-color: #B01F66;
        color: white;
        border: 1px solid black;
        padding: 8px;
    }

    .ewg-theme table td {
        background-color: #089BC9;
        color: black;
        border: 1px solid black;
        padding: 8px;
    }

    .ewg-theme table tr:hover td {
        background-color: #33AEE0;
    }

    .ewg-theme .search-button {
        background-color: #089BC9;
        color: white;
        border: none;
        border-radius: 4px;
        padding: 10px 20px;
        cursor: pointer;
        transition: background-color 0.3s ease;
    }

    .ewg-theme .search-button:hover {
        background-color: #B01F66;
        color: white;
    }

    .ewg-theme .search-button:active {
        background-color: #B01F66; /* Keep the hover color */
        color: white;
        transform: scale(0.98); /* Slightly shrink the button */
    }

    .ewg-theme .search-button:disabled {
        background-color: #D1D1D1;
        color: #8C8C8C;
        opacity: 0.6;
        cursor: default;
    }
</style>
