<script setup>
import { ref, computed, onMounted } from "vue";
import Konva from "konva";

const stage = ref(null);
const stageConfig = ref({
    width: window.innerWidth - 280,
    height: window.innerHeight,
    draggable: true,
});

const lineDefinitions = [
    { id: "main", name: "Core Flow", color: "#007aff" },
    { id: "admin", name: "Admin Layer", color: "#af52de" },
];

const stations = ref([
    { id: "1", name: "ENTRY", x: 100, y: 300, lines: ["main"] },
    { id: "2", name: "VALIDATE", x: 300, y: 300, lines: ["main"] },
    {
        id: "3",
        name: "HUB",
        x: 500,
        y: 250,
        lines: ["main", "admin"],
        interchange: true,
    },
    { id: "4", name: "PROCESS", x: 700, y: 150, lines: ["main"] },
    { id: "5", name: "SHIP", x: 900, y: 150, lines: ["main"] },
    { id: "6", name: "LOGGER", x: 450, y: 500, lines: ["admin"] },
    { id: "7", name: "ARCHIVE", x: 850, y: 450, lines: ["admin"] },
]);

// Helper to determine if a station is the start or end of a specific line
const isTerminal = (stationId, lineId) => {
    const lineStations = stations.value
        .filter((s) => s.lines.includes(lineId))
        .sort((a, b) => a.x - b.x);
    return (
        lineStations[0].id === stationId ||
        lineStations[lineStations.length - 1].id === stationId
    );
};

const generatedPaths = computed(() => {
    return lineDefinitions.map((def) => {
        const lineStations = stations.value
            .filter((s) => s.lines.includes(def.id))
            .sort((a, b) => a.x - b.x);

        if (lineStations.length < 2) return { ...def, path: "" };

        let d = `M ${lineStations[0].x} ${lineStations[0].y}`;
        for (let i = 0; i < lineStations.length - 1; i++) {
            const p1 = lineStations[i];
            const p2 = lineStations[i + 1];

            const dx = Math.abs(p2.x - p1.x);
            const dy = Math.abs(p2.y - p1.y);

            // Orthogonal/Octilinear logic
            if (dx > dy) {
                const transitionX = p1.x + dy * Math.sign(p2.x - p1.x);
                d += ` L ${transitionX} ${p2.y} L ${p2.x} ${p2.y}`;
            } else {
                const transitionY = p1.y + dx * Math.sign(p2.y - p1.y);
                d += ` L ${p2.x} ${transitionY} L ${p2.x} ${p2.y}`;
            }
        }
        return { ...def, path: d };
    });
});

const handleDragMove = (e, station) => {
    const gridSize = 25;
    // Calculate snapped coordinates
    const newX = Math.round(e.target.x() / gridSize) * gridSize;
    const newY = Math.round(e.target.y() / gridSize) * gridSize;

    // Update the reactive station object
    station.x = newX;
    station.y = newY;

    // Manually set the node position to prevent "shimmering" during drag
    e.target.x(newX);
    e.target.y(newY);
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

    const newPos = {
        x: pointer.x - mousePointTo.x * newScale,
        y: pointer.y - mousePointTo.y * newScale,
    };
    stageInstance.position(newPos);
};

const flyTo = (targetX, targetY) => {
    const stageInstance = stage.value.getStage();
    stageInstance.to({
        duration: 0.6,
        easing: Konva.Easings.EaseInOut,
        scaleX: 1.5,
        scaleY: 1.5,
        x: stageConfig.value.width / 2 - targetX * 1.5,
        y: stageConfig.value.height / 2 - targetY * 1.5,
    });
};

const handleMouseEnter = () => (document.body.style.cursor = "pointer");
const handleMouseLeave = () => (document.body.style.cursor = "default");

// Handle window resizing
onMounted(() => {
    window.addEventListener("resize", () => {
        stageConfig.value.width = window.innerWidth - 280;
        stageConfig.value.height = window.innerHeight;
    });
});
</script>

<template>
    <div class="layout">
        <aside class="sidebar">
            <h3>Process Metro Pro</h3>
            <div
                v-for="line in lineDefinitions"
                :key="line.id"
                class="legend-item"
            >
                <div
                    class="line-color"
                    :style="{ background: line.color }"
                ></div>
                <span class="line-name">{{ line.name }}</span>
            </div>
        </aside>

        <main class="map-container" ref="container">
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
                    <v-group
                        v-for="s in stations"
                        :key="s.id"
                        :config="{
                            x: s.x,
                            y: s.y,
                            draggable: true /* Enable dragging */,
                        }"
                        @dragmove="handleDragMove($event, s)"
                        @click="flyTo(s.x, s.y)"
                        @mouseenter="handleMouseEnter"
                        @mouseleave="handleMouseLeave"
                    >
                        <!-- Background Shadow Circle -->
                        <v-circle
                            :config="{
                                radius: 12 /* Changed from r to radius */,
                                fill: 'black',
                                opacity: 0.3,
                            }"
                        />

                        <!-- Standard Station -->
                        <v-circle
                            v-if="!s.interchange"
                            :config="{
                                radius: 8,
                                fill: 'white',
                                stroke: '#222',
                                strokeWidth: 2,
                                shadowColor: 'black',
                                shadowBlur: 2,
                                shadowOffset: { x: 1, y: 1 },
                                shadowOpacity: 0.5,
                            }"
                        />
                        <v-rect
                            v-else
                            :config="{
                                x: -10,
                                y: -20,
                                width: 20,
                                height: 40,
                                cornerRadius: 10,
                                fill: 'white',
                                stroke: '#222',
                                strokeWidth: 3,
                            }"
                        />
                        <v-text
                            :config="{
                                text: s.name,
                                y: 25,
                                align: 'center',
                                width: 100,
                                x: -50,
                                fill: '#aaa',
                                fontSize: 12,
                                fontStyle: 'bold',
                            }"
                        />
                    </v-group>
                </v-layer>
            </v-stage>
        </main>
    </div>
</template>

<style>
/* Global reset to ensure the map fills the screen */
body,
html {
    margin: 0;
    padding: 0;
    height: 100%;
    /*background: #121212;*/
    overflow: hidden;
    font-family: sans-serif;
}

.layout {
    display: flex;
    width: 100vw;
    height: 100vh;
}

.sidebar {
    width: 280px;
    background: #1e1e1e;
    border-right: 1px solid #333;
    padding: 20px;
    color: white;
    flex-shrink: 0;
}

.legend-item {
    display: flex;
    align-items: center;
    margin-bottom: 10px;
}

.line-color {
    width: 12px;
    height: 12px;
    border-radius: 50%;
    margin-right: 10px;
}

.map-container {
    flex-grow: 1;
    /*background: #000;*/
    height: 100%;
}
</style>
