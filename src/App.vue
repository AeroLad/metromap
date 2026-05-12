<script setup>
import { ref, computed, onMounted } from "vue";
import Konva from "konva";

import * as d3 from "d3";

const SNAP_SIZE = 25;

const snapToGrid = (value) => Math.round(value / SNAP_SIZE) * SNAP_SIZE;

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
        x: snapToGrid(150), // ← Snap initial position
        y: snapToGrid(150),
        lines: [],
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
    const connectedLines = station.lines;
    if (connectedLines.length === 0) return 0;

    let vectors = [];

    connectedLines.forEach((lineId) => {
        const lStations = stations.value.filter((s) =>
            s.lines.includes(lineId),
        );
        const idx = lStations.findIndex((s) => s.id === station.id);

        if (lStations[idx - 1])
            vectors.push({
                x: station.x - lStations[idx - 1].x,
                y: station.y - lStations[idx - 1].y,
            });
        if (lStations[idx + 1])
            vectors.push({
                x: lStations[idx + 1].x - station.x,
                y: lStations[idx + 1].y - station.y,
            });
    });

    // Average the vectors to find the "Main Flow"
    const avgX = vectors.reduce((a, b) => a + b.x, 0) / vectors.length;
    const avgY = vectors.reduce((a, b) => a + b.y, 0) / vectors.length;

    let angle = Math.atan2(avgY, avgX) * (180 / Math.PI);

    // Snapping to 45 degrees is essential for the "Professional" look
    let snapped = Math.round(angle / 45) * 45;

    // Return + 90 because capsules are perpendicular to the tracks
    return snapped + 90;
};

const generatedPaths = computed(() => {
    const GAP = 12;
    // We still use our "Topological Sort" to keep the lines from clashing
    const lineOrder = lineDefinitions.value.map((l) => l.id).sort();

    const lineGenerator = d3
        .line()
        .x((d) => d.x)
        .y((d) => d.y)
        .curve(d3.curveLinear); // We will manually inject elbows for octilinear

    return lineDefinitions.value.map((line) => {
        const lStations = stations.value.filter((s) =>
            s.lines.includes(line.id),
        );
        if (lStations.length < 2) return { ...line, path: "" };

        let backbonePoints = [];

        // 1. Generate Octilinear Backbone
        for (let i = 0; i < lStations.length - 1; i++) {
            const s1 = lStations[i];
            const s2 = lStations[i + 1];
            backbonePoints.push({ x: s1.x, y: s1.y });

            // Create the elbow (Octilinear logic)
            const dx = s2.x - s1.x;
            const dy = s2.y - s1.y;
            if (Math.abs(dx) !== Math.abs(dy) && dx !== 0 && dy !== 0) {
                if (Math.abs(dx) > Math.abs(dy)) {
                    backbonePoints.push({
                        x: s1.x + (dx - (dy > 0 ? dy : -dy)),
                        y: s1.y,
                    });
                } else {
                    backbonePoints.push({
                        x: s1.x,
                        y: s1.y + (dy - (dx > 0 ? dx : -dx)),
                    });
                }
            }
            if (i === lStations.length - 2)
                backbonePoints.push({ x: s2.x, y: s2.y });
        }

        // 2. Use D3 to calculate the offset path
        // We calculate the offset for the line based on its global slot
        const slotIdx = lineOrder.indexOf(line.id);
        const offsetDist = (slotIdx - (lineOrder.length - 1) / 2) * GAP;

        // Transform backbone points to offset points using D3's vector logic
        const offsetPoints = backbonePoints.map((p, i) => {
            const prev = backbonePoints[i - 1] || p;
            const next = backbonePoints[i + 1] || p;

            // D3 approach: compute the normal at each point
            // This is where D3's internal math prevents the "kinks"
            const angle =
                i === 0
                    ? Math.atan2(next.y - p.y, next.x - p.x) + Math.PI / 2
                    : Math.atan2(p.y - prev.y, p.x - prev.x) + Math.PI / 2;

            return {
                x: p.x + Math.cos(angle) * offsetDist,
                y: p.y + Math.sin(angle) * offsetDist,
            };
        });

        return {
            ...line,
            path: lineGenerator(offsetPoints), // D3 handles the "M x y L x y" string generation
        };
    });
});

const handleDragMove = (e, station) => {
    const node = e.target;
    const snappedX = snapToGrid(node.x());
    const snappedY = snapToGrid(node.y());

    // Update Konva node directly for smooth visual feedback
    node.position({ x: snappedX, y: snappedY });

    // Sync reactive state
    station.x = snappedX;
    station.y = snappedY;
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
                            // lineCap: 'round',
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
                                width: 10 + s.lines.length * 12,
                                height: 12,
                                // Center the rotation point
                                offsetX: (10 + s.lines.length * 12) / 2,
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
