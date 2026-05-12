<script setup>
import { ref, computed, onMounted } from "vue";

const SNAP_SIZE = 12;
const snapToGrid = (value) => Math.round(value / SNAP_SIZE) * SNAP_SIZE;

const stage = ref(null);
const stageConfig = ref({
    width: window.innerWidth - 320,
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
        x: snapToGrid(150),
        y: snapToGrid(150),
        lines: [],
    });
};

const deleteStation = (index) => stations.value.splice(index, 1);
const deleteLine = (id) => {
    lineDefinitions.value = lineDefinitions.value.filter((l) => l.id !== id);
    stations.value.forEach((s) => {
        s.lines = s.lines.filter((lId) => lId !== id);
    });
};

const toggleLineForStation = (station, lineId) => {
    const index = station.lines.indexOf(lineId);
    if (index > -1) station.lines.splice(index, 1);
    else station.lines.push(lineId);
};

// --- ROTATION (AXIAL CIRCULAR MEAN) ---

const getStationRotation = (station) => {
    if (station.lines.length === 0) return 0;

    const angles = [];

    station.lines.forEach((lineId) => {
        const lStations = stations.value.filter((s) =>
            s.lines.includes(lineId),
        );
        const idx = lStations.findIndex((s) => s.id === station.id);

        if (lStations[idx - 1]) {
            const prev = lStations[idx - 1];
            angles.push(Math.atan2(station.y - prev.y, station.x - prev.x));
        }
        if (lStations[idx + 1]) {
            const next = lStations[idx + 1];
            angles.push(Math.atan2(next.y - station.y, next.x - station.x));
        }
    });

    if (angles.length === 0) return 0;
    if (angles.length === 1) {
        const deg = (angles[0] * 180) / Math.PI;
        return Math.round(deg / 45) * 45 + 90;
    }

    // Axial circular mean: work with doubled angles so 0° and 180° are treated as the same axis
    const doubled = angles.map((a) => a * 2);
    const sinSum = doubled.reduce((s, a) => s + Math.sin(a), 0);
    const cosSum = doubled.reduce((s, a) => s + Math.cos(a), 0);

    let meanAxis;
    const EPS = 1e-9;
    if (Math.hypot(sinSum, cosSum) < EPS) {
        // Vectors cancel (e.g. perfect 90° crossing). Fallback to arithmetic bisector.
        const avg = angles.reduce((a, b) => a + b, 0) / angles.length;
        meanAxis = (avg * 180) / Math.PI;
    } else {
        const meanDoubled = Math.atan2(
            sinSum / doubled.length,
            cosSum / doubled.length,
        );
        meanAxis = ((meanDoubled / 2) * 180) / Math.PI;
    }

    // Perpendicular to the mean axis, snapped to octilinear grid
    return Math.round((meanAxis + 90) / 45) * 45;
};

// --- PATH GENERATION (PER-SEGMENT LOCAL OFFSETS) ---

const generatedPaths = computed(() => {
    const GAP = 10;
    const EPS = 1e-9;

    // ==================== GEOMETRY UTILITIES ====================

    function eq(a, b) {
        return Math.abs(a - b) < EPS;
    }

    function dist(p1, p2) {
        const dx = p1.x - p2.x;
        const dy = p1.y - p2.y;
        return Math.sqrt(dx * dx + dy * dy);
    }

    function len(v) {
        return Math.sqrt(v.x * v.x + v.y * v.y);
    }

    function normalize(v) {
        const l = len(v);
        return l < EPS ? { x: 0, y: 0 } : { x: v.x / l, y: v.y / l };
    }

    /** Left-hand normal (perpendicular, pointing left of direction) */
    function normal(v) {
        const n = normalize(v);
        return { x: -n.y, y: n.x };
    }

    /** Solve 2×2 linear system */
    function solve2x2(a1, b1, c1, a2, b2, c2) {
        const det = a1 * b2 - b1 * a2;
        if (Math.abs(det) < EPS) return null;
        return {
            t: (c1 * b2 - b1 * c2) / det,
            s: (a1 * c2 - c1 * a2) / det,
        };
    }

    // ==================== OCTILINEAR ROUTING ====================

    function octilinearPath(x1, y1, x2, y2) {
        const dx = x2 - x1,
            dy = y2 - y1;
        const adx = Math.abs(dx),
            ady = Math.abs(dy);
        const sx = Math.sign(dx) || 1,
            sy = Math.sign(dy) || 1;

        const candidates = [];

        if (dx === 0 || dy === 0 || adx === ady) {
            candidates.push([
                { x: x1, y: y1 },
                { x: x2, y: y2 },
            ]);
        }

        function add2Segment(bx, by) {
            const p1 = { x: x1, y: y1 },
                p2 = { x: bx, y: by },
                p3 = { x: x2, y: y2 };

            if (
                (eq(p1.x, p2.x) && eq(p1.y, p2.y)) ||
                (eq(p2.x, p3.x) && eq(p2.y, p3.y))
            )
                return;

            const d1x = p2.x - p1.x,
                d1y = p2.y - p1.y;
            const d2x = p3.x - p2.x,
                d2y = p3.y - p2.y;
            const a1 = Math.abs(d1x),
                a1y = Math.abs(d1y);
            const a2 = Math.abs(d2x),
                a2y = Math.abs(d2y);

            const valid1 = eq(a1, 0) || eq(a1y, 0) || eq(a1, a1y);
            const valid2 = eq(a2, 0) || eq(a2y, 0) || eq(a2, a2y);

            if (valid1 && valid2) candidates.push([p1, p2, p3]);
        }

        add2Segment(x2, y1);
        add2Segment(x1, y2);

        if (adx > ady) {
            add2Segment(x2 - sx * ady, y1);
            add2Segment(x1 + sx * ady, y2);
        }
        if (ady > adx) {
            add2Segment(x1, y2 - sy * adx);
            add2Segment(x2, y1 + sy * adx);
        }

        if (candidates.length === 0) {
            const midX = (x1 + x2) / 2;
            candidates.push([
                { x: x1, y: y1 },
                { x: midX, y: y1 },
                { x: midX, y: y2 },
                { x: x2, y: y2 },
            ]);
        }

        function pathLength(path) {
            let l = 0;
            for (let i = 1; i < path.length; i++)
                l += dist(path[i - 1], path[i]);
            return l;
        }

        candidates.sort((a, b) => {
            const d = pathLength(a) - pathLength(b);
            return Math.abs(d) > EPS ? d : a.length - b.length;
        });

        return candidates[0];
    }

    function pruneCollinear(points) {
        if (points.length < 3) return points;
        const out = [points[0]];
        for (let i = 1; i < points.length - 1; i++) {
            const a = out[out.length - 1],
                b = points[i],
                c = points[i + 1];
            const d1x = b.x - a.x,
                d1y = b.y - a.y;
            const d2x = c.x - b.x,
                d2y = c.y - b.y;
            if (Math.abs(d1x * d2y - d1y * d2x) > EPS) out.push(b);
        }
        out.push(points[points.length - 1]);
        return out;
    }

    // ==================== PARALLEL OFFSET ====================

    function parallelOffset(points, offsetDist) {
        if (points.length < 2 || Math.abs(offsetDist) < EPS) {
            return points.map((p) => ({ ...p }));
        }

        const result = [];
        const maxMiterRatio = 4;

        for (let i = 0; i < points.length; i++) {
            const p = points[i],
                prev = points[i - 1],
                next = points[i + 1];

            if (!prev) {
                const d = { x: next.x - p.x, y: next.y - p.y };
                const n = normal(d);
                result.push({
                    x: p.x + n.x * offsetDist,
                    y: p.y + n.y * offsetDist,
                });
            } else if (!next) {
                const d = { x: p.x - prev.x, y: p.y - prev.y };
                const n = normal(d);
                result.push({
                    x: p.x + n.x * offsetDist,
                    y: p.y + n.y * offsetDist,
                });
            } else {
                const d1 = { x: p.x - prev.x, y: p.y - prev.y };
                const d2 = { x: next.x - p.x, y: next.y - p.y };
                const l1 = len(d1),
                    l2 = len(d2);

                if (l1 < EPS || l2 < EPS) {
                    result.push({ x: p.x, y: p.y });
                    continue;
                }

                const n1 = normal(d1);
                const n2 = normal(d2);

                const rhsX = d1.x + offsetDist * (n2.x - n1.x);
                const rhsY = d1.y + offsetDist * (n2.y - n1.y);
                const sol = solve2x2(d1.x, -d2.x, rhsX, d1.y, -d2.y, rhsY);

                if (sol) {
                    const mx = prev.x + offsetDist * n1.x + sol.t * d1.x;
                    const my = prev.y + offsetDist * n1.y + sol.t * d1.y;
                    const miterDist = dist({ x: mx, y: my }, p);

                    if (miterDist <= Math.abs(offsetDist) * maxMiterRatio) {
                        result.push({ x: mx, y: my });
                        continue;
                    }
                }

                const avgX = n1.x + n2.x,
                    avgY = n1.y + n2.y;
                const avgLen = len({ x: avgX, y: avgY });
                if (avgLen > EPS) {
                    const scale =
                        Math.min(
                            miterDist || Math.abs(offsetDist),
                            Math.abs(offsetDist) * maxMiterRatio,
                        ) / Math.abs(offsetDist);
                    result.push({
                        x: p.x + (avgX / avgLen) * offsetDist * scale,
                        y: p.y + (avgY / avgLen) * offsetDist * scale,
                    });
                } else {
                    result.push({
                        x: p.x + n1.x * offsetDist,
                        y: p.y + n1.y * offsetDist,
                    });
                }
            }
        }

        return result;
    }

    // ==================== SVG PATH ====================

    function pointsToPath(points) {
        if (!points.length) return "";
        let d = `M ${points[0].x.toFixed(2)} ${points[0].y.toFixed(2)}`;
        for (let i = 1; i < points.length; i++) {
            d += ` L ${points[i].x.toFixed(2)} ${points[i].y.toFixed(2)}`;
        }
        return d;
    }

    // ==================== MAIN (PER-SEGMENT OFFSETS) ====================

    return lineDefinitions.value.map((line) => {
        const lStations = stations.value.filter((s) =>
            s.lines.includes(line.id),
        );
        if (lStations.length < 2) return { ...line, path: "" };

        let allPoints = [];

        for (let i = 0; i < lStations.length - 1; i++) {
            const s1 = lStations[i];
            const s2 = lStations[i + 1];

            // Determine which lines actually run through BOTH stations of this segment
            const sharingLines = lineDefinitions.value
                .filter((l) => {
                    const stns = stations.value.filter((s) =>
                        s.lines.includes(l.id),
                    );
                    return (
                        stns.some((s) => s.id === s1.id) &&
                        stns.some((s) => s.id === s2.id)
                    );
                })
                .sort((a, b) => a.id.localeCompare(b.id));

            const localIdx = sharingLines.findIndex((l) => l.id === line.id);
            const localCount = sharingLines.length;
            const offsetDist = (localIdx - (localCount - 1) / 2) * GAP;

            // Build and offset this segment independently
            const seg = octilinearPath(s1.x, s1.y, s2.x, s2.y);
            const offsetSeg = parallelOffset(seg, offsetDist);

            if (i === 0) {
                allPoints.push(...offsetSeg);
            } else {
                // If the offset changed between segments, the path naturally shifts
                // at the station. The station symbol covers the junction.
                allPoints.push(...offsetSeg.slice(1));
            }
        }

        return { ...line, path: pointsToPath(allPoints) };
    });
});

const handleDragMove = (e, station) => {
    const node = e.target;
    const snappedX = snapToGrid(node.x());
    const snappedY = snapToGrid(node.y());
    node.position({ x: snappedX, y: snappedY });
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
                            draggable: true,
                            rotation: getStationRotation(s),
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
                                width: 10 + s.lines.length * 12,
                                height: 12,
                                offsetX: (10 + s.lines.length * 12) / 2,
                                offsetY: 6,
                                cornerRadius: 12,
                                fill: 'white',
                                stroke: '#000',
                                strokeWidth: 2,
                                shadowColor: 'rgba(0,0,0,0.2)',
                                shadowBlur: 2,
                                shadowOffset: { x: 1, y: 1 },
                            }"
                        />

                        <!-- Label: counter-rotated so it's always upright -->
                        <v-text
                            :config="{
                                text: s.name,
                                y: 20,
                                align: 'center',
                                width: 120,
                                x: -60,
                                fill: '#222',
                                fontSize: 11,
                                fontStyle: 'bold',
                                rotation: -getStationRotation(s),
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
