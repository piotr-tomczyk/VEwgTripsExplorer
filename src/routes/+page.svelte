<script lang="ts">
    import ewgData from '../static/ewg.json';
    import ewgAircrafts from '../static/ewgAircrafts.json';
    import  { type Route, SearchModes, type Trip } from '../types/types';
    import { FilterUtils } from '../utils/filters';
    import { TripService } from '../services/trip-service';

    let selectedCallsign: string | null = null;
    let hoursLimit: number = 0;
    let legNumber: number = 2;
    const hoursLimitOptions: number[] = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
    let selectedAircraft: string | null = null;
    let generatedAircraftType: string | null = null;
    let selectedDeparture: Route | null = null;
    let selectedDestination: Route | null = null;
    let generatedDeparture: string | null = null;
    let foundTrip: Trip | null = null;
    let isStartAndReturnAtBase: boolean = false;
    let isGeneratingRoute: boolean = false;
    let isError: boolean = false;

    const selectedSearchMode: SearchModes = SearchModes.trips;
    const legCountOptions: number[] = [2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 20, 30, 50, 100];

    $: aircrafts = selectedCallsign ? ewgAircrafts[selectedCallsign] : ewgAircrafts['EWG'];

    $: allRoutes = ewgData.routes.sort((a, b) => FilterUtils.sortAlphabetically(a.icao, b.icao));

    $: routes = getFilteredRoute(selectedCallsign, hoursLimit, selectedAircraft);

    $: departureRoutes = isStartAndReturnAtBase ? FilterUtils.filterRoutesNotFromBase(routes) : routes;

    $: totalTurnaroundTimeInSeconds = foundTrip ? ((foundTrip as Trip).legs.length - 1) * 35 * 60 : 0;

    $: totalTripTime = getTotalTripTime(foundTrip, allRoutes, totalTurnaroundTimeInSeconds);

    $: totalTurnaroundTime = getTotalTurnaroundTime(totalTurnaroundTimeInSeconds);

    $: availableCallsigns = FilterUtils.getAllCallsigns(allRoutes);


    function getFilteredRoute(selectedCallsign, hoursLimit, selectedAircraft) {
        const routeFilteredByCallsing = selectedCallsign
            //@ts-ignore
            ? FilterUtils.filterDataByCallsign(allRoutes, selectedCallsign)
            : allRoutes;

        const routeFilteredByHours = hoursLimit !== 0
            ? FilterUtils.filterDataByHours(routeFilteredByCallsing, hoursLimit)
            : routeFilteredByCallsing;

        const routesFilteredByAircraft = selectedAircraft
            ? FilterUtils.filterDataByAircraft(routeFilteredByHours, selectedAircraft)
            : routeFilteredByHours;

        return routesFilteredByAircraft;
    }

    function getTotalTripTime(foundTrip, allRoutes, totalTurnaroundTimeInSeconds) {
        if (foundTrip?.legs?.length) {
            const tripTimeInSeconds = foundTrip?.legs?.reduce((currentValue, leg) => {
                const legTime = allRoutes?.find(
                    (route) => route.icao === leg.start
                )?.destinations?.find(
                    (destination) => destination.icao === leg.end
                )?.time || '';

                const [hours, minutes, seconds] = legTime.split(':').map(Number);
                return hours * 3600 + minutes * 60 + seconds + currentValue;
            }, 0);

            const tripAndTournaroundTime = tripTimeInSeconds + totalTurnaroundTimeInSeconds;

            const hours = Math.floor(tripAndTournaroundTime / 3600);
            const minutes = Math.floor((tripAndTournaroundTime % 3600) / 60);
            const seconds = tripAndTournaroundTime % 60;

            return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
        }
        return '';
    }

    function getTotalTurnaroundTime(totalTurnaroundTimeInSeconds, ) {
        if (foundTrip) {
            const hours = Math.floor(totalTurnaroundTimeInSeconds / 3600);
            const minutes = Math.floor((totalTurnaroundTimeInSeconds % 3600) / 60);
            const seconds = totalTurnaroundTimeInSeconds % 60;

            return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
        }
    }

    function calculateLegs() {
        if (isGeneratingRoute) {
            return;
        }

        isError = false;

        switch (selectedSearchMode) {
            case SearchModes.legs:
                console.log('Legs');
                break;
            case SearchModes.trips:
                const isSelectedAircraft = !!selectedAircraft;
                selectedAircraft ??=  getRandomAircraft();
                const startAirport = selectedDeparture?.icao ?? getRandomDepartureAirport()?.icao;
                const destinationAirport = selectedDestination?.icao ?? startAirport;

                generatedAircraftType = null;
                foundTrip = null;
                try {
                    isGeneratingRoute = true;

                    const tripsData = generateTrip(startAirport, destinationAirport);

                    if (tripsData?.trip?.legs?.length && selectedCallsign) {
                        localStorage.setItem(selectedCallsign, JSON.stringify(tripsData));
                    }

                    foundTrip = tripsData?.trip ?? null;
                    generatedAircraftType = tripsData?.aircraftType ?? null;
                    generatedDeparture = destinationAirport;
                    if (!tripsData?.trip?.legs?.length) {
                        isError = true;
                    }
                } catch (error) {
                    isError = true;
                    console.log({ error });
                } finally {
                    if (!isSelectedAircraft) {
                        selectedAircraft = null;
                    }
                    isGeneratingRoute = false;
                }
                break;
            default:
                console.log('Default');
        }
    }

    function generateTrip(startAirport: string, destinationAirport: string) {
        const desiredHours = hoursLimit;
        const desiredLegs = legNumber;

        let tripsData = {
            aircraftType: '',
            trip: {
                destination: '',
                legs: [],
            }
        };

        const tripService = new TripService();
        if (isStartAndReturnAtBase) {
            const isOddLeg = desiredLegs % 2 !== 0;
            let numberOfEvenLegs = Math.floor(desiredLegs / 2) - (isOddLeg ? 1 : 0);

            for (let j = 0; j < numberOfEvenLegs; j++) {
                const generatedData = generateLegs(
                    tripService,
                    routes,
                    startAirport,
                    destinationAirport,
                    3,
                    selectedAircraft || undefined,
                    desiredHours
                );

                if (generatedData.trip) {
                    tripsData.aircraftType = generatedData.aircraftType;
                    tripsData.trip.destination = destinationAirport;
                    //@ts-ignore
                    tripsData.trip.legs = tripsData.trip.legs.concat(generatedData.trip.legs);
                }
            }

            if (isOddLeg) {
                const generatedData = generateLegs(
                    tripService,
                    routes,
                    startAirport,
                    destinationAirport,
                    4,
                    selectedAircraft || undefined,
                    desiredHours
                );

                if (generatedData.trip) {
                    tripsData.aircraftType = generatedData.aircraftType;
                    tripsData.trip.destination = destinationAirport;
                    //@ts-ignore
                    tripsData.trip.legs = tripsData.trip.legs.concat(generatedData.trip.legs);
                }
            }
        } else {
            //@ts-ignore
            tripsData = generateLegs(
                tripService,
                routes,
                startAirport,
                destinationAirport,
                desiredLegs + 1,
                selectedAircraft || undefined,
                desiredHours
            );
        }
        return tripsData;
    }

    function generateLegs(
        tripService: TripService,
        routes: Route[],
        startAirport: string,
        destinationAirport: string,
        legs: number,
        aircraftType: string | undefined,
        hoursLimit: number
    ): { trip: Trip | null, aircraftType: string } {
        for (let i = 0; i < 50; i++) {
            const departureAirport = i === 0 ? startAirport : getRandomDepartureAirport()?.icao;
            //@ts-ignore
            const generatedData = tripService.findTrip(
                routes,
                departureAirport,
                destinationAirport,
                legs,
                aircraftType,
                hoursLimit
            );

            if (generatedData.trip) {
                return generatedData;
            }
        }
        return { trip: null, aircraftType: '' };
    }

    function getRandomDepartureAirport() {
        return departureRoutes[Math.floor(Math.random() * departureRoutes?.length)];
    }

    function getRandomAircraft() {
        const aircraftsToProcess = selectedDeparture
            ? selectedDeparture?.aircrafts.filter(
                (aircraft) => aircrafts.find((aircraftType) => aircraftType === aircraft))
            : aircrafts;
        return aircraftsToProcess?.[Math.floor(Math.random() * aircraftsToProcess?.length)];
    }
</script>

<div class="popup-body ewg-theme">
    <div class="form-container">
        <div>
            <span> Airline: </span>
            <select bind:value={selectedCallsign}>
                {#each availableCallsigns as callsign (callsign)}
                    <option value="{callsign}">
                        { callsign }
                    </option>
                {/each}
            </select>
        </div>
        <div>
            <span> Aircraft: </span>
            {#if aircrafts?.length}
                <select bind:value="{selectedAircraft}">
                    {#if aircrafts?.length}
                        <option value="{ null }" />
                    {/if}
                    {#each aircrafts as aircraft (aircraft)}
                        <option value="{aircraft}">
                            {aircraft}
                        </option>
                    {/each}
                </select>
            {/if}
        </div>
        <div>
            <span> Departure: </span>
            <select bind:value={selectedDeparture}>
                {#if routes?.length}
                    <option value="{null}" />
                {/if}
                {#each departureRoutes as airport (airport)}
                    <option value="{airport}">
                        { airport.icao } ( { airport.name } )
                    </option>
                {/each}
            </select>
            <span style="display: flex; align-items: center; margin-top: 5px;">
                <label for="returnToBase">
                    Start and return at base:
                </label>
                <input
                    type="checkbox"
                    id="return-to-base"
                    name="returnToBase"
                    checked="{isStartAndReturnAtBase}"
                    on:click="{() => { isStartAndReturnAtBase = !isStartAndReturnAtBase }}" />
            </span>
        </div>
        {#if selectedDeparture && !isStartAndReturnAtBase}
            <div>
                <span> Destination: </span>
                <select bind:value="{selectedDestination}">
                    {#if routes?.length}
                        <option value="{null}" />
                    {/if}
                    {#each routes as airport (airport)}
                        <option value="{airport}">
                            { airport.icao } ( { airport.name } )
                        </option>
                    {/each}
                </select>
            </div>
        {/if}
        <div>
            <span> Leg number: </span>
            <select bind:value="{legNumber}">
                {#each legCountOptions as leg (leg)}
                    <option value="{leg}">
                        { leg }
                    </option>
                {/each}
            </select>
        </div>
        <div>
            <span> Hour Limit per leg: </span>
            <select bind:value="{hoursLimit}">
                {#each hoursLimitOptions as hour (hour)}
                    <option value="{hour}">
                        { hour > 0 ? hour : 'No limit' }
                    </option>
                {/each}
            </select>
        </div>
    </div>
    <button
        class="search-button"
        disabled="{isGeneratingRoute}"
        on:click="{calculateLegs}">
        Search
    </button>

    {#if isGeneratingRoute}
        <span>
            Generating Route...
        </span>
    {/if}
    {#if foundTrip?.legs?.length}
        <div>
            <h2>Found Trip</h2>
            <p>
                Aircraft:
                <span class="generated-aircraft-type">
                    { generatedAircraftType }
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
                {#each foundTrip?.legs as leg, index (leg)}
                    <tr>
                        <td> { index + 1 } </td>
                        <td> { leg.start } </td>
                        <td> { leg.end } </td>
                        <td>
                            {
                                allRoutes?.find(
                                    (route) => route.icao === leg.start
                                )?.destinations?.find(
                                        (destination) => destination.icao === leg.end
                                )?.time
                            }
                        </td>
                    </tr>
                {/each}
                <tr>
                    {#if totalTripTime?.includes('NaN') && !totalTurnaroundTime?.includes('NaN')}
                        <td colspan="4">
                            Total trip time:
                            <span style="font-weight: bold">
                                { totalTripTime }
                            </span>
                            (includes { totalTurnaroundTime } of turnaroud)
                        </td>
                    {/if}
                </tr>
                </tbody>
            </table>
        </div>
    {:else if generatedAircraftType || isError}
        <div class="error-message">
            {#if routes?.length === 0}
                <span>
                    No possible trip found for selected parameters. Try changing parameters.
                </span>
            {:else if generatedAircraftType}
                <span>
                        No trip found for generated aircraft:
                    { generatedAircraftType }  and { generatedDeparture } departure airport,
                        it might take a few attempts. Try changing parameters or search again.
                </span>
            {:else}
            <span>
                    No trip found for selected parameters. Try changing parameters or search again.
            </span>
            {/if}
        </div>
    {/if}
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
