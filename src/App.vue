<script setup>
import { ref, computed, onMounted, onUnmounted, watch } from "vue";

const SNAP = 12;
const SIDEBAR = 280;
const STUB = 24;
const GAP = 10;
const EPS = 1e-9;
const snap = (v) => Math.round(v / SNAP) * SNAP;

// --- STATE ---
const stage = ref(null);
const stageConfig = ref({
    width: window.innerWidth - SIDEBAR,
    height: window.innerHeight,
    draggable: true,
});

const debugGrid = ref(false);
const fileInput = ref(null);

const linesExpanded = ref(true);
const stationsExpanded = ref(true);

const lineDefinitions = ref([
    { id: crypto.randomUUID(), name: "Main Transit", color: "#3b82f6" },
    { id: crypto.randomUUID(), name: "Service Loop", color: "#8b5cf6" },
]);

const stations = ref([
    {
        id: crypto.randomUUID(),
        name: "Central Terminal",
        x: snap(120),
        y: snap(300),
        lines: [lineDefinitions.value[0].id],
    },
    {
        id: crypto.randomUUID(),
        name: "North Gate",
        x: snap(360),
        y: snap(300),
        lines: [lineDefinitions.value[0].id],
    },
]);

// --- PERSISTENCE ---
const saveJSON = () => {
    const data = { lines: lineDefinitions.value, stations: stations.value };
    const blob = new Blob([JSON.stringify(data, null, 2)], {
        type: "application/json",
    });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = `metro-map-${new Date().toISOString().slice(0, 10)}.json`;
    a.click();
    URL.revokeObjectURL(url);
};

const loadJSON = (e) => {
    const file = e.target.files[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = (ev) => {
        try {
            const data = JSON.parse(ev.target.result);
            if (data.lines?.length) lineDefinitions.value = data.lines;
            if (data.stations?.length) stations.value = data.stations;
        } catch {
            alert("Invalid map file");
        }
        e.target.value = "";
    };
    reader.readAsText(file);
};

const newMap = () => {
    if (!confirm("Clear current map?")) return;
    lineDefinitions.value = [];
    stations.value = [];
};

// --- MUTATIONS ---
const addLine = () => {
    const colors = [
        "#3b82f6",
        "#8b5cf6",
        "#ef4444",
        "#10b981",
        "#f59e0b",
        "#ec4899",
        "#06b6d4",
        "#6366f1",
    ];
    lineDefinitions.value.push({
        id: crypto.randomUUID(),
        name: `Line ${lineDefinitions.value.length + 1}`,
        color: colors[lineDefinitions.value.length % colors.length],
    });
};

const addStation = () => {
    stations.value.push({
        id: crypto.randomUUID(),
        name: "Station",
        x: snap(stageConfig.value.width / 2),
        y: snap(stageConfig.value.height / 2),
        lines: [],
    });
};

const deleteStation = (i) => stations.value.splice(i, 1);
const deleteLine = (id) => {
    lineDefinitions.value = lineDefinitions.value.filter((l) => l.id !== id);
    stations.value.forEach((s) => {
        s.lines = s.lines.filter((lid) => lid !== id);
    });
};

const moveLine = (i, dir) => {
    const j = i + dir;
    if (j < 0 || j >= lineDefinitions.value.length) return;
    const arr = [...lineDefinitions.value];
    const [item] = arr.splice(i, 1);
    arr.splice(j, 0, item);
    lineDefinitions.value = arr;
};

const toggleLine = (station, lineId) => {
    const i = station.lines.indexOf(lineId);
    if (i > -1) station.lines.splice(i, 1);
    else station.lines.push(lineId);
};

// --- GEOMETRY ---
function dominantAngle(station) {
    const dirs = [];
    station.lines.forEach((lid) => {
        const list = stations.value.filter((s) => s.lines.includes(lid));
        const idx = list.findIndex((s) => s.id === station.id);
        if (idx > 0) {
            const p = list[idx - 1];
            dirs.push(
                Math.round(
                    (Math.atan2(station.y - p.y, station.x - p.x) * 180) /
                        Math.PI /
                        45,
                ) * 45,
            );
        } else if (idx === 0 && list.length > 1) {
            const n = list[1];
            dirs.push(
                Math.round(
                    (Math.atan2(n.y - station.y, n.x - station.x) * 180) /
                        Math.PI /
                        45,
                ) * 45,
            );
        }
    });
    if (!dirs.length) return 0;
    const counts = {};
    dirs.forEach((a) => {
        const k = ((a % 360) + 360) % 360;
        counts[k] = (counts[k] || 0) + 1;
    });
    return parseFloat(
        Object.keys(counts).reduce((a, b) => (counts[a] > counts[b] ? a : b)),
    );
}

const stationRotation = (s) => {
    if (s.lines.length <= 1) return 0;
    return Math.round((dominantAngle(s) + 90) / 45) * 45;
};

const labelConfig = (s, i) => {
    const rot = stationRotation(s);
    let textRot = -rot;
    textRot = ((textRot % 360) + 360) % 360;
    if (textRot > 180) textRot -= 360;
    if (textRot > 90) textRot -= 180;
    if (textRot < -90) textRot += 180;

    const side = i % 2 === 0 ? 1 : -1;
    const w = Math.max(80, s.name.length * 7 + 14);
    return {
        text: s.name.toUpperCase(),
        x: side * (20 + w / 2) - w / 2,
        y: -6,
        align: "center",
        width: w,
        fill: "#1e293b",
        fontSize: 11,
        fontFamily: "Inter, system-ui, sans-serif",
        letterSpacing: 0.5,
        fontStyle: "600",
        rotation: textRot,
    };
};

// --- PATH ENGINE ---
const normal = (v) => {
    const l = Math.hypot(v.x, v.y);
    return l < EPS ? { x: 0, y: 0 } : { x: -v.y / l, y: v.x / l };
};

function octi(x1, y1, x2, y2) {
    const dx = x2 - x1,
        dy = y2 - y1;
    const sx = Math.sign(dx) || 1,
        sy = Math.sign(dy) || 1;
    const adx = Math.abs(dx),
        ady = Math.abs(dy);
    if (adx > ady)
        return [
            { x: x1, y: y1 },
            { x: x1 + sx * (adx - ady), y: y1 },
            { x: x2, y: y2 },
        ];
    if (ady > adx)
        return [
            { x: x1, y: y1 },
            { x: x1, y: y1 + sy * (ady - adx) },
            { x: x2, y: y2 },
        ];
    return [
        { x: x1, y: y1 },
        { x: x2, y: y2 },
    ];
}

function offset(points, off) {
    if (!off || points.length < 2) return points;
    const out = [];
    for (let i = 0; i < points.length; i++) {
        const prev = points[i - 1] || points[i];
        const curr = points[i];
        const next = points[i + 1] || points[i];
        const v1 = { x: curr.x - prev.x, y: curr.y - prev.y };
        const v2 = { x: next.x - curr.x, y: next.y - curr.y };
        const n1 = normal(v1),
            n2 = normal(v2);
        let nx, ny;
        if (i === 0) {
            nx = n2.x * off;
            ny = n2.y * off;
        } else if (i === points.length - 1) {
            nx = n1.x * off;
            ny = n1.y * off;
        } else {
            nx = (n1.x + n2.x) * off;
            ny = (n1.y + n2.y) * off;
            const len = Math.hypot(nx, ny);
            if (len > EPS) {
                const sc = Math.abs(off) / len;
                nx *= sc;
                ny *= sc;
            }
        }
        out.push({ x: curr.x + nx, y: curr.y + ny });
    }
    return out;
}

function stub(station, target) {
    if (station.lines.length <= 1) return null;
    // Use the actual direction toward the target so outgoing lines
    // fan out naturally instead of being forced to the station's
    // global perpendicular axis.
    const toTarget = Math.atan2(target.y - station.y, target.x - station.x);
    const dir = Math.round(toTarget / (Math.PI / 4)) * (Math.PI / 4);
    return {
        x: snap(station.x + Math.cos(dir) * STUB),
        y: snap(station.y + Math.sin(dir) * STUB),
    };
}

const generatedPaths = computed(() => {
    return lineDefinitions.value.map((line) => {
        const list = stations.value.filter((s) => s.lines.includes(line.id));
        const segments = [];
        for (let i = 0; i < list.length - 1; i++) {
            const a = list[i],
                b = list[i + 1];
            const shared = lineDefinitions.value.filter(
                (ld) => a.lines.includes(ld.id) && b.lines.includes(ld.id),
            );
            const off =
                (shared.findIndex((ld) => ld.id === line.id) -
                    (shared.length - 1) / 2) *
                GAP;

            const sa = stub(a, b);
            const sb = stub(b, a);
            const start = sa || { x: a.x, y: a.y };
            const end = sb || { x: b.x, y: b.y };
            const oct = octi(start.x, start.y, end.x, end.y);

            const path = [];
            if (sa) path.push({ x: a.x, y: a.y }, sa);
            path.push(
                ...oct.slice(sa ? 1 : 0, sb ? oct.length - 1 : oct.length),
            );
            if (sb) path.push(sb, { x: b.x, y: b.y });

            const shifted = offset(path, off);
            const f = (n) => Math.round(n * 10) / 10;
            const d =
                `M ${f(shifted[0].x)} ${f(shifted[0].y)} ` +
                shifted
                    .slice(1)
                    .map((p) => `L ${f(p.x)} ${f(p.y)}`)
                    .join(" ");
            segments.push(d);
        }
        return { ...line, segments };
    });
});

// --- INTERACTION ---
const onDrag = (e, s) => {
    const node = e.target;
    s.x = snap(node.x());
    s.y = snap(node.y());
    node.position({ x: s.x, y: s.y });
};

const onResize = () => {
    stageConfig.value.width = window.innerWidth - SIDEBAR;
    stageConfig.value.height = window.innerHeight;
};

const onKey = (e) => {
    if (e.ctrlKey && (e.key === "s" || e.key === "S")) {
        e.preventDefault();
        saveJSON();
    }
};

// --- LIFECYCLE ---
watch(
    [lineDefinitions, stations],
    () => {
        localStorage.setItem(
            "processMetro",
            JSON.stringify({
                lines: lineDefinitions.value,
                stations: stations.value,
            }),
        );
    },
    { deep: true },
);

onMounted(() => {
    const saved = localStorage.getItem("processMetro");
    if (saved) {
        try {
            const data = JSON.parse(saved);
            if (data.lines?.length) lineDefinitions.value = data.lines;
            if (data.stations?.length) stations.value = data.stations;
        } catch {
            /* ignore */
        }
    }
    window.addEventListener("resize", onResize);
    window.addEventListener("keydown", onKey);
});

onUnmounted(() => {
    window.removeEventListener("resize", onResize);
    window.removeEventListener("keydown", onKey);
});
</script>

<template>
    <div class="layout">
        <aside class="sidebar">
            <header class="app-header">
                <div class="logo-area">
                    <div class="logo-icon"></div>
                    <h1>Process Metro <span class="badge">PRO</span></h1>
                </div>
                <div class="toolbar">
                    <button class="btn-e" @click="saveJSON">Save</button>
                    <label class="btn-e">
                        Load
                        <input
                            ref="fileInput"
                            type="file"
                            accept=".json"
                            @change="loadJSON"
                            hidden
                        />
                    </label>
                    <button class="btn-e btn-ghost" @click="newMap">New</button>
                </div>
                <label class="debug-switch">
                    <input type="checkbox" v-model="debugGrid" />
                    <span class="switch-label">Show Grid</span>
                </label>
            </header>

            <div class="sidebar-body">
                <section class="panel" :class="{ collapsed: !linesExpanded }">
                    <div
                        class="panel-header"
                        @click="linesExpanded = !linesExpanded"
                    >
                        <div class="header-title">
                            <span class="chevron"></span>
                            <h2>Transit Lines</h2>
                        </div>
                        <button @click.stop="addLine" class="btn-add">+</button>
                    </div>

                    <div class="panel-content">
                        <div
                            v-for="(line, i) in lineDefinitions"
                            :key="line.id"
                            class="card"
                        >
                            <div class="card-row">
                                <div class="color-picker-wrapper">
                                    <input
                                        type="color"
                                        v-model="line.color"
                                        class="color-circle"
                                    />
                                </div>
                                <input
                                    type="text"
                                    v-model="line.name"
                                    class="input-inline"
                                />
                            </div>
                            <div class="card-actions">
                                <button
                                    @click="moveLine(i, -1)"
                                    :disabled="i === 0"
                                >
                                    ↑
                                </button>
                                <button
                                    @click="moveLine(i, 1)"
                                    :disabled="i === lineDefinitions.length - 1"
                                >
                                    ↓
                                </button>
                                <button
                                    @click="deleteLine(line.id)"
                                    class="danger"
                                >
                                    ×
                                </button>
                            </div>
                        </div>
                    </div>
                </section>

                <section
                    class="panel"
                    :class="{ collapsed: !stationsExpanded }"
                >
                    <div
                        class="panel-header"
                        @click="stationsExpanded = !stationsExpanded"
                    >
                        <div class="header-title">
                            <span class="chevron"></span>
                            <h2>Stations</h2>
                        </div>
                        <button @click.stop="addStation" class="btn-add">
                            +
                        </button>
                    </div>

                    <div class="panel-content">
                        <div
                            v-for="(s, i) in stations"
                            :key="s.id"
                            class="card"
                        >
                            <div class="card-row header-row">
                                <input
                                    type="text"
                                    v-model="s.name"
                                    class="input-inline weight-600"
                                />
                                <button
                                    @click="deleteStation(i)"
                                    class="btn-delete"
                                >
                                    Delete
                                </button>
                            </div>
                            <div class="chips">
                                <button
                                    v-for="line in lineDefinitions"
                                    :key="line.id"
                                    @click="toggleLine(s, line.id)"
                                    class="chip-circle"
                                    :class="{
                                        active: s.lines.includes(line.id),
                                    }"
                                    :style="
                                        s.lines.includes(line.id)
                                            ? {
                                                  backgroundColor: line.color,
                                                  borderColor: line.color,
                                              }
                                            : {}
                                    "
                                >
                                    {{ line.name[0] }}
                                </button>
                            </div>
                        </div>
                    </div>
                </section>
            </div>

            <footer class="sidebar-footer">
                <span>v3.0.0-enterprise</span>
                <span>Ctrl+S to save</span>
            </footer>
        </aside>

        <main class="map-container">
            <div class="canvas-bg" :class="{ hidden: !debugGrid }"></div>
            <v-stage ref="stage" :config="stageConfig">
                <v-layer>
                    <template v-for="line in generatedPaths" :key="line.id">
                        <v-path
                            v-for="(pathData, i) in line.segments"
                            :key="i"
                            :config="{
                                data: pathData,
                                stroke: line.color,
                                strokeWidth: 8,
                                lineCap: 'round',
                                lineJoin: 'round',
                                shadowColor: 'rgba(0,0,0,0.08)',
                                shadowBlur: 6,
                                shadowOffset: { x: 0, y: 3 },
                            }"
                        />
                    </template>
                    <v-group
                        v-for="(s, i) in stations"
                        :key="s.id"
                        :config="{
                            x: s.x,
                            y: s.y,
                            draggable: true,
                            rotation: stationRotation(s),
                            onDragMove: (e) => onDrag(e, s),
                        }"
                    >
                        <v-circle
                            v-if="s.lines.length <= 1"
                            :config="{
                                radius: 7,
                                fill: 'white',
                                stroke: '#1e293b',
                                strokeWidth: 3,
                            }"
                        />
                        <v-rect
                            v-else
                            :config="{
                                width: 12 + s.lines.length * 8,
                                height: 14,
                                offsetX: (12 + s.lines.length * 8) / 2,
                                offsetY: 7,
                                cornerRadius: 7,
                                fill: 'white',
                                stroke: '#1e293b',
                                strokeWidth: 3,
                            }"
                        />
                        <v-text :config="labelConfig(s, i)" />
                    </v-group>
                </v-layer>
            </v-stage>
        </main>
    </div>
</template>

<style scoped>
.layout {
    display: flex;
    width: 100vw;
    height: 100vh;
    background: #ffffff;
    font-family: "Inter", system-ui, sans-serif;
    color: #0f172a;
    overflow: hidden;
}

/* --- SIDEBAR STRUCTURE --- */
.sidebar {
    width: 280px;
    background: #f8fafc;
    border-right: 1px solid #e2e8f0;
    display: flex;
    flex-direction: column;
    z-index: 10;
    flex-shrink: 0;
}

.app-header {
    padding: 16px;
    background: #ffffff;
    border-bottom: 1px solid #e2e8f0;
    display: flex;
    flex-direction: column;
    gap: 12px;
}

.sidebar-body {
    flex: 1;
    overflow-y: auto;
    scrollbar-width: thin;
    scrollbar-color: #cbd5e1 transparent;
}

/* --- BUTTONS (Enterprise Style) --- */
.toolbar {
    display: flex;
    gap: 6px;
}

.btn-e {
    flex: 1;
    height: 28px;
    background: #ffffff;
    border: 1px solid #e2e8f0;
    color: #1e293b;
    border-radius: 6px;
    font-size: 11px;
    font-weight: 600;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    box-shadow: 0 1px 2px rgba(0, 0, 0, 0.05);
    transition: all 0.15s ease;
}

.btn-e:hover {
    background: #f1f5f9;
    border-color: #cbd5e1;
}

.btn-ghost {
    background: transparent;
    border-color: transparent;
    box-shadow: none;
    color: #64748b;
}

.btn-ghost:hover {
    background: #f1f5f9;
}

.btn-add {
    background: transparent;
    border: none;
    color: #3b82f6;
    font-size: 16px;
    cursor: pointer;
    padding: 0 4px;
    line-height: 1;
}

/* --- PANELS (Collapsible) --- */
.panel {
    border-bottom: 1px solid #e2e8f0;
}

.panel-header {
    padding: 8px 16px;
    display: flex;
    justify-content: space-between;
    align-items: center;
    cursor: pointer;
    user-select: none;
}

.panel-header:hover {
    background: #f1f5f9;
}

.header-title {
    display: flex;
    align-items: center;
    gap: 6px;
}

h2 {
    font-size: 10px;
    text-transform: uppercase;
    letter-spacing: 0.06em;
    color: #94a3b8;
    font-weight: 700;
    margin: 0;
}

.chevron {
    width: 0;
    height: 0;
    border-left: 4px solid transparent;
    border-right: 4px solid transparent;
    border-top: 5px solid #94a3b8;
    transition: transform 0.2s ease;
}

.collapsed .chevron {
    transform: rotate(-90deg);
}

.panel-content {
    padding: 0 16px 16px;
    display: flex;
    flex-direction: column;
    gap: 8px;
}

.collapsed .panel-content {
    display: none;
}

/* --- CARDS & INPUTS --- */
.card {
    background: #ffffff;
    border: 1px solid #e2e8f0;
    border-radius: 6px;
    padding: 8px;
    box-shadow: 0 1px 2px rgba(0, 0, 0, 0.02);
}

.card-row {
    display: flex;
    align-items: center;
    gap: 8px;
}

.input-inline {
    flex: 1;
    background: transparent;
    border: none;
    font-size: 12px;
    color: #1e293b;
    padding: 2px 0;
    border-bottom: 1px solid transparent;
}

.input-inline:focus {
    outline: none;
    border-bottom-color: #3b82f6;
}

.weight-600 {
    font-weight: 600;
}

/* --- LINE INDICATORS (Circles) --- */
.color-picker-wrapper {
    width: 18px;
    height: 18px;
    position: relative;
}

.color-circle {
    appearance: none;
    width: 18px;
    height: 18px;
    border-radius: 50%;
    border: 2px solid #e2e8f0;
    cursor: pointer;
    padding: 0;
    background: none;
    overflow: hidden;
}

.color-circle::-webkit-color-swatch-wrapper {
    padding: 0;
}
.color-circle::-webkit-color-swatch {
    border: none;
    border-radius: 50%;
}

.chip-circle {
    width: 22px;
    height: 22px;
    border-radius: 50%;
    border: 1px solid #e2e8f0;
    background: transparent;
    color: #94a3b8;
    font-size: 10px;
    font-weight: 700;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: all 0.15s ease;
}

.chip-circle.active {
    color: #ffffff;
    border-color: transparent;
}

/* --- UTILS --- */
.card-actions {
    margin-top: 8px;
    display: flex;
    justify-content: flex-end;
    gap: 4px;
}

.card-actions button {
    width: 22px;
    height: 22px;
    background: #f8fafc;
    border: 1px solid #e2e8f0;
    border-radius: 4px;
    color: #64748b;
    cursor: pointer;
    font-size: 11px;
    display: flex;
    align-items: center;
    justify-content: center;
}

.card-actions button:hover:not(:disabled) {
    background: #f1f5f9;
    color: #0f172a;
}

.btn-delete {
    background: transparent;
    border: none;
    color: #94a3b8;
    font-size: 10px;
    cursor: pointer;
    font-weight: 600;
    padding: 0;
}

.btn-delete:hover {
    color: #ef4444;
}
.danger:hover {
    color: #ef4444 !important;
    border-color: #ef4444 !important;
}

/* --- CANVAS & FOOTER --- */
.map-container {
    flex: 1;
    position: relative;
    background: #fcfcfc;
}
.canvas-bg {
    position: absolute;
    inset: 0;
    background-image: radial-gradient(#e2e8f0 1px, transparent 1px);
    background-size: 24px 24px;
}
.hidden {
    opacity: 0;
}

.sidebar-footer {
    padding: 8px 16px;
    background: #ffffff;
    border-top: 1px solid #e2e8f0;
    font-size: 10px;
    color: #94a3b8;
    display: flex;
    justify-content: space-between;
}
</style>
