<template>
    <v-container fluid>
        <v-row>
            <v-col cols="12" md="6">
                <v-card outlined elevation="2">
                    <v-card-text>
                        <v-text-field
                            v-if="!!startSystem"
                            label="Start system"
                            append-icon="mdi-close"
                            readonly
                            outlined
                            :value="startSystem.name"
                            @click:append="startSystem = null"
                        ></v-text-field>
                        <v-text-field
                            v-else
                            label="Start system"
                            v-model="startInput"
                            outlined
                            clearable
                            :error-messages="startInputErrors"
                        >
                            <v-icon
                                slot="append"
                                color="green"
                                @click="setstart"
                            >
                                mdi-check
                            </v-icon>
                        </v-text-field>
                        <v-input
                            v-for="system of intermediate"
                            :key="system.name"
                            readonly
                            append-icon="mdi-close"
                        >
                            {{ system.name }}
                        </v-input>
                        <v-text-field
                            label="Add system"
                            v-model="systemInput"
                            clearable
                            :error-messages="systemInputErrors"
                        >
                            <v-icon
                                slot="append"
                                color="green"
                                @click="addSystem"
                            >
                                mdi-plus
                            </v-icon>
                        </v-text-field>
                    </v-card-text>
                </v-card>
            </v-col>
            <v-col cols="12" md="6">
                <v-card outlined elevation="2">
                    <v-toolbar dense elevation="1">
                        <v-toolbar-title>Result</v-toolbar-title>
                        <v-spacer></v-spacer>
                        <v-btn text @click="clear">Clear</v-btn>
                        <v-btn text @click="reveal">Reveal</v-btn>
                    </v-toolbar>
                    <v-list-item
                        three-line
                        v-for="system of route"
                        :key="system.name"
                    >
                        <v-list-item-content>
                            <v-list-item-title>
                                {{ system.name }}
                            </v-list-item-title>
                            <v-list-item-subtitle>
                                Distance: {{ +system.distance.toFixed(2) }}
                            </v-list-item-subtitle>
                            <v-list-item-subtitle>
                                From start: {{ +system.fromStart.toFixed(2) }}
                            </v-list-item-subtitle>
                        </v-list-item-content>
                    </v-list-item>
                </v-card>
            </v-col>
        </v-row>
    </v-container>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';

interface Location {
    x: number;
    y: number;
    z: number;
}

interface System {
    name: string;
    location: Location;
}

interface Revealed {
    name: string;
    distance: number;
    fromStart: number;
}

@Component
export default class Route extends Vue {

    startSystem: System | null = null;
    intermediate: System[] = [];
    startInput = '';
    startInputErrors: string[] = [];
    systemInput = '';
    systemInputErrors: string[] = [];
    route: Revealed[] = [];

    setstart(): void {
        const system = this.startInput?.trim();
        if (!system) return;
        this.startInput = '';
        this.startInputErrors = [];

        if (this.intermediate.findIndex((s) => s.name === system) !== -1) {
            this.startInputErrors.push('Already exists');
            return;
        }

        this.$http
            .get(
                `https://www.edsm.net/api-v1/system?systemName=${system}&showCoordinates=1`
            )
            .then((response) => {
                if (!response.data.name) {
                    this.startInputErrors.push('Not found in EDSM database');
                    return;
                }
                this.startSystem = {
                    name: response.data.name,
                    location: response.data.coords,
                };
            })
            .catch((err) => {
                this.startInputErrors.push(err.toString());
            });
    }

    addSystem(): void {
        const system = this.systemInput?.trim();
        if (!system) return;
        this.systemInput = '';
        this.systemInputErrors = [];

        if (this.intermediate.findIndex((s) => s.name === system) !== -1) {
            this.systemInputErrors.push('Already exists');
            return;
        }

        this.$http
            .get(
                `https://www.edsm.net/api-v1/system?systemName=${system}&showCoordinates=1`
            )
            .then((response) => {
                if (!response.data.name) {
                    this.systemInputErrors.push('Not found in EDSM database');
                    return;
                }
                this.intermediate.push({
                    name: response.data.name,
                    location: response.data.coords,
                });
            })
            .catch((err) => {
                this.systemInputErrors.push(err.toString());
            });
    }

    clear(): void {
        this.startSystem = null;
        this.intermediate = [];
        this.route = [];
    }

    reveal(): void {
        if (!this.startSystem) return;
        if (this.intermediate.length < 2) return;

        function distance3d(from: Location, to: Location) {
            return Math.sqrt(
                Math.pow(Math.abs(to.x - from.x), 2) +
                    Math.pow(Math.abs(to.y - from.y), 2) +
                    Math.pow(Math.abs(to.z - from.z), 2)
            );
        }

        // Calculating distance matrix between intermediate systems
        const distMatrix = Array(this.intermediate.length)
            .fill(null)
            .map(() => Array(this.intermediate.length).fill(0));
        for (const [fx, from] of this.intermediate.entries()) {
            for (const [tx, to] of this.intermediate.entries()) {
                distMatrix[fx][tx] = distance3d(from.location, to.location);
            }
        }

        // Calculating distance from start system to intermediates
        const start = this.startSystem;
        const systems = this.intermediate.map((s) => {
            return {
                name: s.name,
                location: s.location,
                fromStart: distance3d(start.location, s.location),
            };
        });

        const route: Revealed[] = [];
        let next = systems[0];
        let clearIndex = 0;

        // Searching first system by minimum distance from start
        for (const [sx, sv] of systems.entries()) {
            if (sv.fromStart < next.fromStart) {
                next = sv;
                clearIndex = sx;
            }
        }

        let nextDistance = next.fromStart;
        while (true) {
            route.push({
                name: next.name,
                distance: nextDistance,
                fromStart: next.fromStart,
            });

            // Remove start and check if any other left
            systems.splice(clearIndex, 1);
            if (systems.length === 0) break;

            // Searching for next nearest intermediate
            const sx = this.intermediate.findIndex((s) => s.name === next.name);
            next = systems[0];
            clearIndex = 0;
            nextDistance =
                distMatrix[sx][
                    this.intermediate.findIndex((s) => s.name === next.name)
                ];
            for (const [cx, cv] of systems.entries()) {
                const nx = this.intermediate.findIndex(
                    (s) => s.name === cv.name
                );
                if (distMatrix[sx][nx] < nextDistance) {
                    nextDistance = distMatrix[sx][nx];
                    next = cv;
                    clearIndex = cx;
                }
            }
        }

        this.route = route;
    }

}
</script>
