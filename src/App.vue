<script setup>
import { ref, computed, onMounted } from "vue";
import Konva from "konva";

const stage = ref(null);
const stageConfig = ref({
    width: window.innerWidth - 320, // Adjusted for slightly wider sidebar
    height: window.innerHeight,
    draggable: true,
});

// --- STATE MANAGEMENT ---

const lineDefinitions = ref([
    { id: "line-" + Date.now(), name: "Core Flow", color: "#007aff" },
    { id: "line-" + (Date.now() + 1), name: "Admin Layer", color: "#af52de" },
]);

const stations = ref([
    {
        id: "1",
        name: "ENTRY",
        x: 100,
        y: 300,
        lines: [lineDefinitions.value[0].id],
        interchange: false,
    },
    {
        id: "2",
        name: "VALIDATE",
        x: 300,
        y: 300,
        lines: [lineDefinitions.value[0].id],
        interchange: false,
    },
]);

// --- METHODS ---

const addLine = () => {
    const id = "line-" + Date.now();
    lineDefinitions.value.push({
        id,
        name: `New Line ${lineDefinitions.value.length + 1}`,
        color: "#" + Math.floor(Math.random() * 16777215).toString(16),
    });
};

const addStation = () => {
    const id = "st-" + Date.now();
    stations.value.push({
        id,
        name: "NEW STATION",
        x: 150,
        y: 150,
        lines: [], // Interchange state is now automatically inferred from this
    });
};

const deleteStation = (index) => stations.value.splice(index, 1);
const deleteLine = (id) => {
    lineDefinitions.value = lineDefinitions.value.filter((l) => l.id !== id);
    // Cleanup stations that were on this line
    stations.value.forEach((s) => {
        s.lines = s.lines.filter((lId) => lId !== id);
    });
};

const toggleLineForStation = (station, lineId) => {
    const index = station.lines.indexOf(lineId);
    if (index > -1) station.lines.splice(index, 1);
    else station.lines.push(lineId);
};

// --- LOGIC (KEEP EXISTING) ---
const getStationRotation = (station) => {
    if (station.lines.length < 2) return 0;

    // Anchor to the first line's flow
    const lineId = station.lines[0];
    const lStations = stations.value.filter((s) => s.lines.includes(lineId));
    const idx = lStations.findIndex((s) => s.id === station.id);

    const prev = lStations[idx - 1];
    const next = lStations[idx + 1];

    // Calculate direction of the line through this station
    let dx = 0,
        dy = 0;
    if (prev && next) {
        dx = next.x - prev.x;
        dy = next.y - prev.y;
    } else if (next) {
        dx = next.x - station.x;
        dy = next.y - station.y;
    } else if (prev) {
        dx = station.x - prev.x;
        dy = station.y - prev.y;
    }

    // Snap to 45-degree increments
    let angle = Math.atan2(dy, dx) * (180 / Math.PI);
    let snapped = Math.round(angle / 45) * 45;

    // Capsule is perpendicular to track
    return snapped + 90;
};

const generatedPaths = computed(() => {
    const GAP = 12;

    return lineDefinitions.value.map((line) => {
        const lStations = stations.value.filter((s) =>
            s.lines.includes(line.id),
        );
        if (lStations.length < 2) return { ...line, path: "" };

        let segments = [];

        // Helper to get a stable offset point at any station
        const getStationOffsetPt = (station) => {
            const rotRad = (getStationRotation(station) * Math.PI) / 180;
            // Use global line definition index so order never flips
            const globalIdx = lineDefinitions.value.findIndex(
                (ld) => ld.id === line.id,
            );
            const total = lineDefinitions.value.length;
            const amt = (globalIdx - (total - 1) / 2) * GAP;

            return {
                x: station.x + Math.cos(rotRad) * amt,
                y: station.y + Math.sin(rotRad) * amt,
            };
        };

        for (let i = 0; i < lStations.length - 1; i++) {
            const s1 = lStations[i];
            const s2 = lStations[i + 1];

            const pStart = getStationOffsetPt(s1);
            const pEnd = getStationOffsetPt(s2);

            // Octilinear Elbow Logic
            const dx = pEnd.x - pStart.x;
            const dy = pEnd.y - pStart.y;
            let mx, my;

            if (Math.abs(dx) > Math.abs(dy)) {
                mx = pStart.x + (Math.abs(dx) - Math.abs(dy)) * Math.sign(dx);
                my = pStart.y;
            } else {
                mx = pStart.x;
                my = pStart.y + (Math.abs(dy) - Math.abs(dx)) * Math.sign(dy);
            }

            if (i === 0) segments.push(`M ${pStart.x} ${pStart.y}`);
            segments.push(`L ${mx} ${my} L ${pEnd.x} ${pEnd.y}`);
        }
        return { ...line, path: segments.join(" ") };
    });
});

const handleDragMove = (e, station) => {
    const gridSize = 25;
    station.x = Math.round(e.target.x() / gridSize) * gridSize;
    station.y = Math.round(e.target.y() / gridSize) * gridSize;
};

const handleWheel = (e) => {
    e.evt.preventDefault();
    const stageInstance = stage.value.getStage();
    const oldScale = stageInstance.scaleX();
    const pointer = stageInstance.getPointerPosition();
    const mousePointTo = {
        x: (pointer.x - stageInstance.x()) / oldScale,
        y: (pointer.y - stageInstance.y()) / oldScale,
    };
    const newScale = e.evt.deltaY < 0 ? oldScale * 1.1 : oldScale / 1.1;
    stageInstance.scale({ x: newScale, y: newScale });
    stageInstance.position({
        x: pointer.x - mousePointTo.x * newScale,
        y: pointer.y - mousePointTo.y * newScale,
    });
};

const handleMouseEnter = () => (document.body.style.cursor = "move");
const handleMouseLeave = () => (document.body.style.cursor = "default");

onMounted(() => {
    window.addEventListener("resize", () => {
        stageConfig.value.width = window.innerWidth - 320;
        stageConfig.value.height = window.innerHeight;
    });
});
</script>

<template>
    <div class="layout">
        <aside class="sidebar">
            <h2>Process Metro Pro</h2>

            <section>
                <div class="section-header">
                    <h3>Lines</h3>
                    <button @click="addLine" class="btn-add">+</button>
                </div>
                <div
                    v-for="line in lineDefinitions"
                    :key="line.id"
                    class="editor-item"
                >
                    <input type="color" v-model="line.color" />
                    <input type="text" v-model="line.name" class="name-input" />
                    <button @click="deleteLine(line.id)" class="btn-del">
                        ×
                    </button>
                </div>
            </section>

            <hr />

            <section>
                <div class="section-header">
                    <h3>Stations</h3>
                    <button @click="addStation" class="btn-add">+</button>
                </div>
                <div
                    v-for="(s, index) in stations"
                    :key="s.id"
                    class="station-editor"
                >
                    <div class="station-row">
                        <input
                            type="text"
                            v-model="s.name"
                            class="name-input"
                        />
                        <!-- <button
                            @click="s.interchange = !s.interchange"
                            :class="{ active: s.interchange }"
                            class="btn-icon"
                        >
                            {{ s.interchange ? "⬥" : "○" }}
                        </button> -->
                        <button @click="deleteStation(index)" class="btn-del">
                            ×
                        </button>
                    </div>
                    <div class="line-chips">
                        <span
                            v-for="line in lineDefinitions"
                            :key="line.id"
                            @click="toggleLineForStation(s, line.id)"
                            :class="{ selected: s.lines.includes(line.id) }"
                            :style="{
                                borderColor: line.color,
                                backgroundColor: s.lines.includes(line.id)
                                    ? line.color
                                    : 'transparent',
                            }"
                        >
                            {{ line.name[0] }}
                        </span>
                    </div>
                </div>
            </section>
        </aside>

        <main class="map-container">
            <v-stage ref="stage" :config="stageConfig" @wheel="handleWheel">
                <v-layer>
                    <!-- Paths -->
                    <v-path
                        v-for="line in generatedPaths"
                        :key="line.id"
                        :config="{
                            data: line.path,
                            stroke: line.color,
                            strokeWidth: 8,
                            lineCap: 'round',
                            lineJoin: 'round',
                        }"
                    />

                    <!-- Nodes -->
                    <!-- Updated Station Group -->
                    <v-group
                        v-for="s in stations"
                        :key="s.id"
                        :config="{
                            x: s.x,
                            y: s.y,
                            draggable: true,
                            rotation: getStationRotation(s), // Dynamic rotation applied here
                        }"
                        @dragmove="handleDragMove($event, s)"
                    >
                        <!-- Standard Station -->
                        <v-circle
                            v-if="s.lines.length <= 1"
                            :config="{
                                radius: 8,
                                fill: 'white',
                                stroke: '#222',
                                strokeWidth: 2.5,
                            }"
                        />

                        <!-- Dynamic Capsule Interchange -->
                        <v-rect
                            v-else
                            :config="{
                                // Calculate dimensions
                                width: 20 + s.lines.length * 12,
                                height: 24,
                                // Center the rotation point
                                offsetX: (20 + s.lines.length * 12) / 2,
                                offsetY: 12,

                                cornerRadius: 12,
                                fill: 'white',
                                stroke: '#000', // Solid black or dark grey
                                strokeWidth: 2,
                                // This makes it pop off the lines
                                shadowColor: 'rgba(0,0,0,0.2)',
                                shadowBlur: 2,
                                shadowOffset: { x: 1, y: 1 },
                            }"
                        />

                        <!-- Label: We counter-rotate the text so it's always readable -->
                        <v-text
                            :config="{
                                text: s.name,
                                y: 20,
                                align: 'center',
                                width: 120,
                                x: -30,
                                fill: '#222',
                                fontSize: 11,
                                fontStyle: 'bold',
                                rotation: -getStationRotation(s), // Keeps text upright
                            }"
                        />
                    </v-group>
                </v-layer>
            </v-stage>
        </main>
    </div>
</template>

<style>
body,
html {
    margin: 0;
    height: 100%;
    overflow: hidden;
    font-family: "Inter", sans-serif;
    background: #f0f0f0;
}

.layout {
    display: flex;
    width: 100vw;
    height: 100vh;
}

.sidebar {
    width: 320px;
    background: #1e1e1e;
    padding: 20px;
    color: #ececec;
    overflow-y: auto;
    box-shadow: 4px 0 10px rgba(0, 0, 0, 0.3);
    z-index: 10;
}

.section-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 15px;
}

.editor-item,
.station-row {
    display: flex;
    align-items: center;
    gap: 8px;
    margin-bottom: 10px;
}

.name-input {
    background: #333;
    border: 1px solid #444;
    color: white;
    padding: 4px 8px;
    border-radius: 4px;
    flex-grow: 1;
}

.line-chips {
    display: flex;
    gap: 4px;
    margin-top: 5px;
    flex-wrap: wrap;
}
.line-chips span {
    font-size: 10px;
    padding: 2px 6px;
    border: 1px solid;
    border-radius: 10px;
    cursor: pointer;
    opacity: 0.6;
}
.line-chips span.selected {
    opacity: 1;
    color: white;
}

.btn-add {
    background: #28a745;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    width: 24px;
    height: 24px;
}
.btn-del {
    background: transparent;
    color: #ff4444;
    border: none;
    cursor: pointer;
    font-size: 18px;
}
.btn-icon {
    background: #444;
    border: none;
    color: white;
    cursor: pointer;
    border-radius: 4px;
    padding: 4px 8px;
}
.btn-icon.active {
    background: #007aff;
}

hr {
    border: 0;
    border-top: 1px solid #444;
    margin: 20px 0;
}

.map-container {
    flex-grow: 1;
    position: relative;
}
</style>
