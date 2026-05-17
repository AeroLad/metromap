<script setup>
import { ref, computed } from "vue";

// --- CONSTANTS ---
const SNAP = 12;
const SIDEBAR = 280;
const GAP = 16;
const CORNER_RADIUS = 12;
const BAR_HALF_HEIGHT = 4;

const snap = (v) => Math.round(v / SNAP) * SNAP;

// --- STATE ---
const lineDefinitions = ref([
    { id: "l1", name: "Central Line", color: "#ef4444" },
    { id: "l2", name: "District Line", color: "#10b981" },
    { id: "l3", name: "Piccadilly", color: "#3b82f6" },
]);

const stations = ref([
    {
        id: "s1",
        name: "Westminster",
        x: 120,
        y: 360,
        lines: ["l1", "l2", "l3"],
        labelPos: "N",
        labelOffsetX: 0,
        labelOffsetY: 0,
    },
    {
        id: "s2",
        name: "Victoria",
        x: 360,
        y: 360,
        lines: ["l1", "l3"],
        labelPos: "S",
        labelOffsetX: 0,
        labelOffsetY: 0,
    },
    {
        id: "s3",
        name: "Embankment",
        x: 504,
        y: 216,
        lines: ["l1", "l2"],
        labelPos: "NE",
        labelOffsetX: 0,
        labelOffsetY: 0,
    },
]);

const stageRef = ref(null);
const fileInput = ref(null);

const stageConfig = ref({
    width: window.innerWidth - SIDEBAR,
    height: window.innerHeight,
    draggable: true,
    onWheel: onStageWheel,
});

// --- LIST MANAGEMENT ---
const moveItem = (list, index, direction) => {
    const newIndex = index + direction;
    if (newIndex < 0 || newIndex >= list.length) return;
    const element = list.splice(index, 1)[0];
    list.splice(newIndex, 0, element);
};

const deleteStation = (id) =>
    (stations.value = stations.value.filter((s) => s.id !== id));
const deleteLine = (id) => {
    lineDefinitions.value = lineDefinitions.value.filter((l) => l.id !== id);
    stations.value.forEach(
        (s) => (s.lines = s.lines.filter((lId) => lId !== id)),
    );
};

// --- SAVE / LOAD ---
function saveToJson() {
    const data = {
        lineDefinitions: JSON.parse(JSON.stringify(lineDefinitions.value)),
        stations: JSON.parse(JSON.stringify(stations.value)),
    };
    const blob = new Blob([JSON.stringify(data, null, 2)], {
        type: "application/json",
    });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = "metro-map.json";
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
}

function triggerLoad() {
    fileInput.value?.click();
}

function loadFromJson(e) {
    const file = e.target.files?.[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = (event) => {
        try {
            const data = JSON.parse(event.target.result);
            if (data.lineDefinitions && Array.isArray(data.lineDefinitions)) {
                lineDefinitions.value = data.lineDefinitions;
            }
            if (data.stations && Array.isArray(data.stations)) {
                stations.value = data.stations.map((s) => ({
                    ...s,
                    lines: Array.isArray(s.lines) ? s.lines : [],
                }));
            }
        } catch {
            alert("Invalid JSON file");
        }
        e.target.value = "";
    };
    reader.readAsText(file);
}

// --- GEOMETRY ENGINE ---

function getOctilinearPoints(p1, p2) {
    const dx = p2.x - p1.x;
    const dy = p2.y - p1.y;
    const adx = Math.abs(dx);
    const ady = Math.abs(dy);
    const sx = Math.sign(dx) || 1;
    const sy = Math.sign(dy) || 1;

    if (adx > ady && ady !== 0)
        return [p1, { x: p1.x + sx * (adx - ady), y: p1.y }, p2];
    if (ady > adx && adx !== 0)
        return [p1, { x: p1.x, y: p1.y + sy * (ady - adx) }, p2];
    return [p1, p2];
}

function offsetPolyline(points, offset) {
    if (offset === 0 || points.length < 2) return points;
    const result = [];
    for (let i = 0; i < points.length; i++) {
        const p = points[i];
        const next = points[i + 1] || p;
        const prev = points[i - 1] || p;

        const v1 = { x: p.x - prev.x, y: p.y - prev.y };
        const v2 = { x: next.x - p.x, y: next.y - p.y };

        const mag1 = Math.hypot(v1.x, v1.y) || 1;
        const mag2 = Math.hypot(v2.x, v2.y) || 1;

        const n1 = { x: -v1.y / mag1, y: v1.x / mag1 };
        const n2 = { x: -v2.y / mag2, y: v2.x / mag2 };

        if (i === 0) {
            result.push({ x: p.x + n2.x * offset, y: p.y + n2.y * offset });
        } else if (i === points.length - 1) {
            result.push({ x: p.x + n1.x * offset, y: p.y + n1.y * offset });
        } else {
            const bisector = { x: n1.x + n2.x, y: n1.y + n2.y };
            const bMag = Math.hypot(bisector.x, bisector.y) || 1;
            const dot = n1.x * n2.x + n1.y * n2.y;
            const miterScale = offset / Math.sqrt(Math.max((1 + dot) / 2, 0.2));
            result.push({
                x: p.x + (bisector.x / bMag) * miterScale,
                y: p.y + (bisector.y / bMag) * miterScale,
            });
        }
    }
    return result;
}

function getSmoothPath(points, radius) {
    if (points.length < 2) return "";
    let d = `M ${points[0].x} ${points[0].y}`;
    for (let i = 1; i < points.length - 1; i++) {
        const pPrev = points[i - 1],
            pCurr = points[i],
            pNext = points[i + 1];
        const v1 = { x: pPrev.x - pCurr.x, y: pPrev.y - pCurr.y },
            v2 = { x: pNext.x - pCurr.x, y: pNext.y - pCurr.y };
        const d1 = Math.hypot(v1.x, v1.y),
            d2 = Math.hypot(v2.x, v2.y);
        const r = Math.min(radius, d1 / 2, d2 / 2);
        const s = {
            x: pCurr.x + (v1.x / d1) * r,
            y: pCurr.y + (v1.y / d1) * r,
        };
        const e = {
            x: pCurr.x + (v2.x / d2) * r,
            y: pCurr.y + (v2.y / d2) * r,
        };
        d += ` L ${s.x} ${s.y} Q ${pCurr.x} ${pCurr.y}, ${e.x} ${e.y}`;
    }
    d += ` L ${points[points.length - 1].x} ${points[points.length - 1].y}`;
    return d;
}

// --- PATH HELPERS ---
function circlePath(x, y, r) {
    return `M ${x + r} ${y} A ${r} ${r} 0 1 1 ${x - r} ${y} A ${r} ${r} 0 1 1 ${x + r} ${y} Z`;
}

// --- GLOBAL TRACK OFFSETS ---
const renderedLines = computed(() => {
    const totalLines = lineDefinitions.value.length;
    return lineDefinitions.value.map((line, globalIndex) => {
        const lineStations = stations.value.filter((s) =>
            s.lines.includes(line.id),
        );
        const paths = [];
        const offsetDist = (globalIndex - (totalLines - 1) / 2) * GAP;

        let fullBasePoints = [];
        for (let i = 0; i < lineStations.length - 1; i++) {
            const s1 = lineStations[i];
            const s2 = lineStations[i + 1];
            const pts = getOctilinearPoints(s1, s2);
            if (i === 0) {
                fullBasePoints.push(...pts);
            } else {
                fullBasePoints.push(...pts.slice(1));
            }
        }

        if (fullBasePoints.length >= 2) {
            const offsetPoints = offsetPolyline(fullBasePoints, offsetDist);
            paths.push(getSmoothPath(offsetPoints, CORNER_RADIUS));
        }

        return { ...line, paths, globalIndex };
    });
});

// --- CUSTOM STATION RENDERER ---
const stationRenderData = computed(() => {
    const totalLines = lineDefinitions.value.length;

    return stations.value.map((s) => {
        const positions = [];
        const stationLineIds = [...s.lines].sort((a, b) => {
            const idxA = lineDefinitions.value.findIndex((l) => l.id === a);
            const idxB = lineDefinitions.value.findIndex((l) => l.id === b);
            return idxA - idxB;
        });

        stationLineIds.forEach((lineId) => {
            const globalIndex = lineDefinitions.value.findIndex(
                (l) => l.id === lineId,
            );
            const offsetDist = (globalIndex - (totalLines - 1) / 2) * GAP;

            const lineStations = stations.value.filter((st) =>
                st.lines.includes(lineId),
            );
            const idx = lineStations.findIndex((st) => st.id === s.id);

            let localPolyline = [];
            let stationPolyIndex = 0;

            if (idx > 0) {
                const prev = lineStations[idx - 1];
                const prevPoints = getOctilinearPoints(prev, s);
                localPolyline.push(...prevPoints);
                stationPolyIndex = prevPoints.length - 1;
            }

            if (idx < lineStations.length - 1) {
                const next = lineStations[idx + 1];
                const nextPoints = getOctilinearPoints(s, next);
                if (localPolyline.length > 0) {
                    localPolyline.push(...nextPoints.slice(1));
                } else {
                    localPolyline.push(...nextPoints);
                    stationPolyIndex = 0;
                }
            }

            const offsetPoly = offsetPolyline(localPolyline, offsetDist);
            const absPos = offsetPoly[stationPolyIndex];

            positions.push({
                x: absPos ? absPos.x - s.x : 0,
                y: absPos ? absPos.y - s.y : 0,
                lineId,
            });
        });

        const isJunction = positions.length > 1;
        const markerRadius = GAP * 0.5;
        const markerStroke = 2.5;

        const singleMarkerPath = !isJunction
            ? circlePath(
                  positions[0]?.x ?? 0,
                  positions[0]?.y ?? 0,
                  markerRadius,
              )
            : "";

        const maxOffset = positions.reduce(
            (max, p) => Math.max(max, Math.hypot(p.x, p.y)),
            0,
        );
        const hitRadius = Math.max(22, maxOffset + markerRadius + 10);

        return {
            station: s,
            positions,
            isJunction,
            markerRadius,
            markerStroke,
            hitRadius,
            singleMarkerPath,
        };
    });
});

// --- GENERATE CUSTOM DUMBBELL CONFIG FOR V-SHAPE ---
const getJunctionShapeConfig = (item) => {
    const R = item.markerRadius;
    const hBase = Math.min(BAR_HALF_HEIGHT, R - 1);
    const pts = item.positions;

    return {
        fill: "white",
        stroke: "#1e293b",
        strokeWidth: item.markerStroke,
        sceneFunc(context, shape) {
            if (pts.length < 2) {
                if (pts.length === 1) {
                    context.beginPath();
                    context.arc(pts[0].x, pts[0].y, R, 0, Math.PI * 2, false);
                    context.closePath();
                    context.fillStrokeShape(shape);
                }
                return;
            }

            const angles = [];
            const gammas = [];
            for (let i = 0; i < pts.length - 1; i++) {
                const p1 = pts[i];
                const p2 = pts[i + 1];
                const dist = Math.hypot(p2.x - p1.x, p2.y - p1.y);
                angles.push(Math.atan2(p2.y - p1.y, p2.x - p1.x));

                const minH =
                    dist < 2 * R
                        ? Math.sqrt(R * R - (dist / 2) * (dist / 2))
                        : 0;
                const hEff = Math.max(hBase, minH);
                gammas.push(Math.asin(Math.min(hEff / R, 0.999)));
            }

            context.beginPath();

            const alpha0 = angles[0];
            const gamma0 = gammas[0];
            context.moveTo(
                pts[0].x + R * Math.cos(alpha0 - gamma0),
                pts[0].y + R * Math.sin(alpha0 - gamma0),
            );

            for (let i = 0; i < pts.length - 1; i++) {
                const alpha = angles[i];
                const gamma = gammas[i];
                const pNext = pts[i + 1];

                context.lineTo(
                    pNext.x + R * Math.cos(alpha - Math.PI + gamma),
                    pNext.y + R * Math.sin(alpha - Math.PI + gamma),
                );

                if (i < pts.length - 2) {
                    const nextAlpha = angles[i + 1];
                    const nextGamma = gammas[i + 1];
                    context.arc(
                        pNext.x,
                        pNext.y,
                        R,
                        alpha - Math.PI + gamma,
                        nextAlpha - nextGamma,
                        false,
                    );
                }
            }

            const alphaLast = angles[angles.length - 1];
            const gammaLast = gammas[gammas.length - 1];
            const pLast = pts[pts.length - 1];
            context.arc(
                pLast.x,
                pLast.y,
                R,
                alphaLast - Math.PI + gammaLast,
                alphaLast + Math.PI - gammaLast,
                false,
            );

            for (let i = pts.length - 2; i >= 0; i--) {
                const alpha = angles[i];
                const gamma = gammas[i];
                const pCurr = pts[i];

                context.lineTo(
                    pCurr.x + R * Math.cos(alpha + gamma),
                    pCurr.y + R * Math.sin(alpha + gamma),
                );

                if (i > 0) {
                    const prevAlpha = angles[i - 1];
                    const prevGamma = gammas[i - 1];
                    context.arc(
                        pCurr.x,
                        pCurr.y,
                        R,
                        alpha + gamma,
                        prevAlpha + Math.PI - prevGamma,
                        false,
                    );
                }
            }

            context.arc(
                pts[0].x,
                pts[0].y,
                R,
                alpha0 + gamma0,
                alpha0 - gamma0,
                false,
            );

            context.closePath();
            context.fillStrokeShape(shape);
        },
    };
};

const getLabelProps = (s) => {
    const baseOffset = s.lines.length > 1 ? 24 : 18;
    const configs = {
        N: { x: 0, y: -baseOffset, align: "center", v: "bottom" },
        S: { x: 0, y: baseOffset, align: "center", v: "top" },
        E: { x: baseOffset + 5, y: 0, align: "left", v: "middle" },
        W: { x: -baseOffset - 5, y: 0, align: "right", v: "middle" },
        NE: { x: baseOffset, y: -baseOffset, align: "left", v: "bottom" },
    };
    const c = configs[s.labelPos] || configs.N;
    return {
        text: s.name.toUpperCase(),
        x: -50,
        y: -10,
        width: 100,
        height: 20,
        align: "center",
        verticalAlign: "middle",
        fontSize: 12,
        fontStyle: "700",
        fill: "#1e293b",
    };
};

const onDrag = (e, s) => {
    s.x = snap(e.target.x());
    s.y = snap(e.target.y());
    e.target.position({ x: s.x, y: s.y });
};

const onLabelDrag = (e, s) => {
    s.labelOffsetX = snap(e.target.x()) - s.x;
    s.labelOffsetY = snap(e.target.y()) - s.y;

    e.target.position({
        x: s.x + s.labelOffsetX,
        y: s.y + s.labelOffsetY,
    });
};

function onStageWheel(e) {
    e.evt.preventDefault();
    if (!e.evt.ctrlKey && !e.evt.metaKey) return;

    const stage = e.target.getStage();
    const oldScale = stage.scaleX();
    const pointer = stage.getPointerPosition();
    const scaleBy = 1.05;

    const newScale = e.evt.deltaY > 0 ? oldScale / scaleBy : oldScale * scaleBy;
    const clamped = Math.max(0.1, Math.min(newScale, 5));

    stage.scale({ x: clamped, y: clamped });

    const mousePointTo = {
        x: (pointer.x - stage.x()) / oldScale,
        y: (pointer.y - stage.y()) / oldScale,
    };

    stage.position({
        x: pointer.x - mousePointTo.x * clamped,
        y: pointer.y - mousePointTo.y * clamped,
    });
}

function zoomIn() {
    const stage = stageRef.value?.getStage();
    if (!stage) return;
    const oldScale = stage.scaleX();
    const newScale = Math.min(oldScale * 1.2, 5);
    const center = { x: stage.width() / 2, y: stage.height() / 2 };
    const mousePointTo = {
        x: (center.x - stage.x()) / oldScale,
        y: (center.y - stage.y()) / oldScale,
    };
    stage.scale({ x: newScale, y: newScale });
    stage.position({
        x: center.x - mousePointTo.x * newScale,
        y: center.y - mousePointTo.y * newScale,
    });
}

function zoomOut() {
    const stage = stageRef.value?.getStage();
    if (!stage) return;
    const oldScale = stage.scaleX();
    const newScale = Math.max(oldScale / 1.2, 0.1);
    const center = { x: stage.width() / 2, y: stage.height() / 2 };
    const mousePointTo = {
        x: (center.x - stage.x()) / oldScale,
        y: (center.y - stage.y()) / oldScale,
    };
    stage.scale({ x: newScale, y: newScale });
    stage.position({
        x: center.x - mousePointTo.x * newScale,
        y: center.y - mousePointTo.y * newScale,
    });
}

function resetZoom() {
    const stage = stageRef.value?.getStage();
    if (!stage) return;
    stage.scale({ x: 1, y: 1 });
    stage.position({ x: 0, y: 0 });
}
</script>

<
<template>
    <div class="layout">
        <aside class="sidebar">
            <header class="app-header">
                <h1>Metro Designer <span class="badge">PRO</span></h1>
            </header>

            <div class="sidebar-toolbar">
                <button @click="saveToJson" class="btn-toolbar">
                    Save JSON
                </button>
                <button @click="triggerLoad" class="btn-toolbar">
                    Load JSON
                </button>
                <input
                    ref="fileInput"
                    type="file"
                    accept=".json"
                    @change="loadFromJson"
                    style="display: none"
                />
            </div>

            <div class="sidebar-body">
                <section class="section">
                    <div class="section-header">
                        <h3>Tracks</h3>
                        <button
                            @click="
                                lineDefinitions.push({
                                    id: `l${Date.now()}`,
                                    name: 'New Track',
                                    color: '#6366f1',
                                })
                            "
                            class="btn-add"
                        >
                            +
                        </button>
                    </div>
                    <div
                        v-for="(line, idx) in lineDefinitions"
                        :key="line.id"
                        class="list-item-ui"
                    >
                        <div class="order-controls">
                            <button
                                @click="moveItem(lineDefinitions, idx, -1)"
                                :disabled="idx === 0"
                            >
                                ▲
                            </button>
                            <button
                                @click="moveItem(lineDefinitions, idx, 1)"
                                :disabled="idx === lineDefinitions.length - 1"
                            >
                                ▼
                            </button>
                        </div>
                        <input
                            type="color"
                            v-model="line.color"
                            class="color-picker"
                        />
                        <input v-model="line.name" class="input-minimal" />
                        <button @click="deleteLine(line.id)" class="btn-del">
                            ×
                        </button>
                    </div>
                </section>

                <section class="section">
                    <div class="section-header">
                        <h3>Stations</h3>
                        <button
                            @click="
                                stations.push({
                                    id: `s${Date.now()}`,
                                    name: 'New Station',
                                    x: 100,
                                    y: 100,
                                    lines: [],
                                    labelPos: 'N',
                                    labelOffsetX: 0,
                                    labelOffsetY: 0,
                                })
                            "
                            class="btn-add"
                        >
                            +
                        </button>
                    </div>
                    <div v-for="(s, idx) in stations" :key="s.id" class="card">
                        <div class="card-top-row">
                            <div class="order-controls horizontal">
                                <button
                                    @click="moveItem(stations, idx, -1)"
                                    :disabled="idx === 0"
                                >
                                    ▲
                                </button>
                                <button
                                    @click="moveItem(stations, idx, 1)"
                                    :disabled="idx === stations.length - 1"
                                >
                                    ▼
                                </button>
                            </div>
                            <input v-model="s.name" class="input-inline" />
                            <button
                                @click="deleteStation(s.id)"
                                class="btn-del-card"
                            >
                                ×
                            </button>
                        </div>

                        <div class="track-toggles">
                            <label
                                v-for="line in lineDefinitions"
                                :key="line.id"
                                class="toggle-pill"
                                :style="{
                                    borderColor: s.lines.includes(line.id)
                                        ? line.color
                                        : '#e2e8f0',
                                }"
                            >
                                <input
                                    type="checkbox"
                                    :value="line.id"
                                    v-model="s.lines"
                                />
                                <span
                                    :style="{
                                        color: s.lines.includes(line.id)
                                            ? line.color
                                            : '#94a3b8',
                                    }"
                                    >{{ line.name.charAt(0) }}</span
                                >
                            </label>
                        </div>
                    </div>
                </section>
            </div>
        </aside>

        <main class="map-container">
            <v-stage ref="stageRef" :config="stageConfig">
                <v-layer>
                    <template v-for="line in renderedLines" :key="line.id">
                        <v-path
                            v-for="(path, i) in line.paths"
                            :key="i"
                            :config="{
                                data: path,
                                stroke: line.color,
                                strokeWidth: 8,
                                lineCap: 'round',
                                lineJoin: 'round',
                            }"
                        />
                    </template>

                    <template
                        v-for="item in stationRenderData"
                        :key="item.station.id"
                    >
                        <v-group
                            :config="{
                                x: item.station.x,
                                y: item.station.y,
                                draggable: true,
                                cursor: 'grab',
                                onDragStart: (e) => {
                                    e.target
                                        .getStage()
                                        .container().style.cursor = 'grabbing';
                                },
                                onDragEnd: (e) => {
                                    e.target
                                        .getStage()
                                        .container().style.cursor = '';
                                },
                                onDragMove: (e) => onDrag(e, item.station),
                            }"
                        >
                            <v-path
                                :config="{
                                    data: circlePath(0, 0, item.hitRadius),
                                    fill: 'transparent',
                                }"
                            />

                            <v-shape
                                v-if="item.isJunction"
                                :config="getJunctionShapeConfig(item)"
                            />

                            <v-path
                                v-if="!item.isJunction"
                                :config="{
                                    data: item.singleMarkerPath,
                                    fill: 'white',
                                    stroke: '#1e293b',
                                    strokeWidth: item.markerStroke,
                                }"
                            />
                        </v-group>

                        <v-group
                            :config="{
                                x:
                                    item.station.x +
                                    (item.station.labelOffsetX || 0),
                                y:
                                    item.station.y +
                                    (item.station.labelOffsetY || 0),
                                draggable: true,
                                cursor: 'grab',
                                onDragStart: (e) => {
                                    e.target
                                        .getStage()
                                        .container().style.cursor = 'grabbing';
                                },
                                onDragEnd: (e) => {
                                    e.target
                                        .getStage()
                                        .container().style.cursor = '';
                                },
                                onDragMove: (e) => onLabelDrag(e, item.station),
                            }"
                        >
                            <v-text :config="getLabelProps(item.station)" />
                        </v-group>
                    </template>
                </v-layer>
            </v-stage>

            <div class="zoom-controls">
                <button @click="zoomIn" class="zoom-btn" title="Zoom In">
                    +
                </button>
                <button @click="zoomOut" class="zoom-btn" title="Zoom Out">
                    −
                </button>
                <button @click="resetZoom" class="zoom-btn" title="Reset">
                    ⟲
                </button>
            </div>
        </main>
    </div>
</template>

<style scoped>
.layout {
    display: flex;
    height: 100vh;
    background: #fcfcfc;
    font-family: "Inter", sans-serif;
    overflow: hidden;
}
.sidebar {
    width: 280px;
    background: #f8fafc;
    border-right: 1px solid #e2e8f0;
    padding: 12px;
    overflow-y: auto;
}
.section {
    margin-bottom: 20px;
}
.section-header h3 {
    font-size: 10px;
    text-transform: uppercase;
    color: #64748b;
    letter-spacing: 0.05em;
}

.list-item-ui {
    display: flex;
    align-items: center;
    gap: 8px;
    margin-bottom: 6px;
    background: white;
    padding: 6px;
    border-radius: 6px;
    border: 1px solid #e2e8f0;
}
.order-controls {
    display: flex;
    flex-direction: column;
    gap: 2px;
}
.order-controls.horizontal {
    flex-direction: row;
    margin-right: 8px;
}
.order-controls button {
    font-size: 8px;
    padding: 2px 4px;
    background: #f1f5f9;
    border: 1px solid #e2e8f0;
    cursor: pointer;
    border-radius: 3px;
}

.card {
    background: white;
    padding: 10px;
    border-radius: 8px;
    border: 1px solid #e2e8f0;
    margin-bottom: 10px;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
}
.card-top-row {
    display: flex;
    align-items: center;
    margin-bottom: 8px;
}
.input-inline {
    border: none;
    font-weight: 800;
    flex: 1;
    font-size: 12px;
    outline: none;
}

.track-toggles {
    display: flex;
    flex-wrap: wrap;
    gap: 4px;
}
.toggle-pill {
    border: 1.5px solid #e2e8f0;
    border-radius: 4px;
    font-size: 9px;
    font-weight: 900;
    cursor: pointer;
    width: 20px;
    height: 20px;
    display: flex;
    align-items: center;
    justify-content: center;
}
.toggle-pill input {
    display: none;
}

.btn-add {
    background: #1e293b;
    color: white;
    border: none;
    width: 20px;
    height: 20px;
    border-radius: 4px;
    cursor: pointer;
}
.btn-del,
.btn-del-card {
    background: transparent;
    color: #cbd5e1;
    border: none;
    cursor: pointer;
    font-size: 14px;
}
.btn-del:hover,
.btn-del-card:hover {
    color: #ef4444;
}

.color-picker {
    width: 18px;
    height: 18px;
    border: none;
    padding: 0;
    background: none;
    cursor: pointer;
    border-radius: 50%;
}
.input-minimal {
    border: none;
    background: transparent;
    font-size: 12px;
    flex: 1;
    outline: none;
    font-weight: 600;
}
.map-container {
    flex: 1;
    position: relative;
    cursor: crosshair;
}
.badge {
    font-size: 10px;
    background: #1e293b;
    color: white;
    padding: 2px 4px;
    border-radius: 4px;
}

.sidebar-toolbar {
    display: flex;
    gap: 8px;
    margin-bottom: 12px;
}
.btn-toolbar {
    flex: 1;
    font-size: 11px;
    font-weight: 600;
    padding: 6px 0;
    border: 1px solid #e2e8f0;
    background: white;
    border-radius: 6px;
    cursor: pointer;
    color: #1e293b;
}
.btn-toolbar:hover {
    background: #f1f5f9;
}

.zoom-controls {
    position: absolute;
    bottom: 20px;
    left: 20px;
    display: flex;
    gap: 8px;
    z-index: 10;
}
.zoom-btn {
    width: 36px;
    height: 36px;
    border-radius: 8px;
    border: 1px solid #e2e8f0;
    background: white;
    color: #1e293b;
    font-size: 18px;
    font-weight: 600;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    box-shadow: 0 2px 6px rgba(0, 0, 0, 0.08);
}
.zoom-btn:hover {
    background: #f1f5f9;
}
</style>
