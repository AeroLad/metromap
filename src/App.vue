<script setup>
import {
    ref,
    computed,
    watch,
    nextTick,
    onMounted,
    onBeforeUnmount,
} from "vue";

// TipTap imports
import { Editor, EditorContent } from "@tiptap/vue-3";
import StarterKit from "@tiptap/starter-kit";

// --- CONSTANTS ---
const SNAP = 12;
const SIDEBAR = 320;
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
        title: "",
        sections: [],
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
        title: "",
        sections: [],
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
        title: "",
        sections: [],
    },
]);

const stageRef = ref(null);
const fileInput = ref(null);
const tracksCollapsed = ref(false);
const stationsCollapsed = ref(false);

const selectedStationId = ref(null);
const rightSidebarCollapsed = ref(true);
const rightSidebarWidth = ref(460);

const selectedStation = computed(() =>
    stations.value.find((s) => s.id === selectedStationId.value),
);

// --- TIPTAP EDITORS PER SECTION ---
const editors = ref(new Map()); // sectionId -> Editor instance

function getOrCreateEditor(sectionId, initialContent) {
    if (editors.value.has(sectionId)) {
        const existing = editors.value.get(sectionId);
        if (existing.isDestroyed) {
            editors.value.delete(sectionId);
        } else {
            return existing;
        }
    }
    const editor = new Editor({
        extensions: [StarterKit],
        content: initialContent || "",
        onUpdate: ({ editor }) => {
            const s = selectedStation.value;
            if (!s) return;
            const section = s.sections.find((sec) => sec.id === sectionId);
            if (section) {
                section.content = editor.getHTML();
            }
        },
    });
    editors.value.set(sectionId, editor);
    return editor;
}

function destroyEditors() {
    editors.value.forEach((editor) => {
        if (!editor.isDestroyed) editor.destroy();
    });
    editors.value.clear();
}

onBeforeUnmount(() => {
    destroyEditors();
});

// --- RIGHT SIDEBAR SECTIONS ---
function addSection() {
    const s = selectedStation.value;
    if (!s) return;
    if (!Array.isArray(s.sections)) s.sections = [];
    s.sections.push({
        id: `sec_${Date.now()}`,
        title: "New Section",
        content: "",
        collapsed: false,
    });
    nextTick(() => {
        const last = s.sections[s.sections.length - 1];
        getOrCreateEditor(last.id, "");
    });
}

function deleteSection(sectionId) {
    const s = selectedStation.value;
    if (!s) return;
    const editor = editors.value.get(sectionId);
    if (editor && !editor.isDestroyed) editor.destroy();
    editors.value.delete(sectionId);
    s.sections = s.sections.filter((sec) => sec.id !== sectionId);
}

function moveSection(index, direction) {
    const s = selectedStation.value;
    if (!s) return;
    const newIndex = index + direction;
    if (newIndex < 0 || newIndex >= s.sections.length) return;
    const el = s.sections.splice(index, 1)[0];
    s.sections.splice(newIndex, 0, el);
}

// --- STAGE CONFIG ---
const stageConfig = computed(() => ({
    width:
        window.innerWidth -
        SIDEBAR -
        (rightSidebarCollapsed.value ? 0 : rightSidebarWidth.value),
    height: window.innerHeight,
    draggable: true,
    onWheel: onStageWheel,
}));

// --- LIST MANAGEMENT ---
const moveItem = (list, index, direction) => {
    const newIndex = index + direction;
    if (newIndex < 0 || newIndex >= list.length) return;
    const element = list.splice(index, 1)[0];
    list.splice(newIndex, 0, element);
};

const deleteStation = (id) => {
    stations.value = stations.value.filter((s) => s.id !== id);
    if (selectedStationId.value === id) closeRightSidebar();
};
const deleteLine = (id) => {
    lineDefinitions.value = lineDefinitions.value.filter((l) => l.id !== id);
    stations.value.forEach(
        (s) => (s.lines = s.lines.filter((lId) => lId !== id)),
    );
};

// --- RIGHT SIDEBAR ---
function selectStation(id) {
    selectedStationId.value = id;
    rightSidebarCollapsed.value = false;
}

function closeRightSidebar() {
    destroyEditors();
    rightSidebarCollapsed.value = true;
    selectedStationId.value = null;
}

function startResizeRight(e) {
    e.preventDefault();
    const startX = e.clientX;
    const startWidth = rightSidebarWidth.value;

    function onMove(ev) {
        const delta = startX - ev.clientX;
        rightSidebarWidth.value = Math.max(
            260,
            Math.min(startWidth + delta, 600),
        );
    }
    function onUp() {
        document.removeEventListener("mousemove", onMove);
        document.removeEventListener("mouseup", onUp);
    }
    document.addEventListener("mousemove", onMove);
    document.addEventListener("mouseup", onUp);
}

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
                destroyEditors();
                stations.value = data.stations.map((s) => ({
                    ...s,
                    lines: Array.isArray(s.lines) ? s.lines : [],
                    title: s.title || "",
                    sections: Array.isArray(s.sections)
                        ? s.sections.map((sec) => ({
                              ...sec,
                              content: sec.content || "",
                              collapsed: sec.collapsed ?? false,
                          }))
                        : [],
                }));
            }
            closeRightSidebar();
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

function circlePath(x, y, r) {
    return `M ${x + r} ${y} A ${r} ${r} 0 1 1 ${x - r} ${y} A ${r} ${r} 0 1 1 ${x + r} ${y} Z`;
}

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
                <div class="app-title">
                    <h1>Metro Designer <span class="badge">PRO</span></h1>
                </div>
                <div class="app-actions">
                    <button class="btn-toolbar" @click="saveToJson">
                        Save JSON
                    </button>
                    <button class="btn-toolbar" @click="triggerLoad">
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
            </header>

            <div class="sidebar-body">
                <!-- Tracks Accordion -->
                <div class="accordion" :class="{ collapsed: tracksCollapsed }">
                    <button
                        class="accordion-header"
                        @click="tracksCollapsed = !tracksCollapsed"
                    >
                        <div class="accordion-title">
                            <span class="accordion-label">Tracks</span>
                            <span class="count-badge">{{
                                lineDefinitions.length
                            }}</span>
                        </div>
                        <svg
                            class="accordion-chevron"
                            width="14"
                            height="14"
                            viewBox="0 0 16 16"
                            fill="none"
                        >
                            <path
                                d="M4 6L8 10L12 6"
                                stroke="currentColor"
                                stroke-width="1.5"
                                stroke-linecap="round"
                                stroke-linejoin="round"
                            />
                        </svg>
                    </button>

                    <div v-show="!tracksCollapsed" class="accordion-content">
                        <div class="item-list">
                            <div
                                v-for="(line, idx) in lineDefinitions"
                                :key="line.id"
                                class="track-row"
                            >
                                <div class="row-grip">
                                    <button
                                        class="btn-icon"
                                        title="Move up"
                                        @click="
                                            moveItem(lineDefinitions, idx, -1)
                                        "
                                        :disabled="idx === 0"
                                    >
                                        ↑
                                    </button>
                                    <button
                                        class="btn-icon"
                                        title="Move down"
                                        @click="
                                            moveItem(lineDefinitions, idx, 1)
                                        "
                                        :disabled="
                                            idx === lineDefinitions.length - 1
                                        "
                                    >
                                        ↓
                                    </button>
                                </div>
                                <input
                                    type="color"
                                    v-model="line.color"
                                    class="color-swatch"
                                    title="Track color"
                                />
                                <input
                                    v-model="line.name"
                                    class="input-row"
                                    placeholder="Track name"
                                />
                                <button
                                    class="btn-icon btn-delete"
                                    title="Delete track"
                                    @click="deleteLine(line.id)"
                                >
                                    ×
                                </button>
                            </div>
                        </div>
                        <button
                            class="btn-add-full"
                            @click="
                                lineDefinitions.push({
                                    id: `l${Date.now()}`,
                                    name: 'New Track',
                                    color: '#6366f1',
                                })
                            "
                        >
                            <span class="btn-add-icon">+</span>
                            <span>Add Track</span>
                        </button>
                    </div>
                </div>

                <!-- Stations Accordion -->
                <div
                    class="accordion"
                    :class="{ collapsed: stationsCollapsed }"
                >
                    <button
                        class="accordion-header"
                        @click="stationsCollapsed = !stationsCollapsed"
                    >
                        <div class="accordion-title">
                            <span class="accordion-label">Stations</span>
                            <span class="count-badge">{{
                                stations.length
                            }}</span>
                        </div>
                        <svg
                            class="accordion-chevron"
                            width="14"
                            height="14"
                            viewBox="0 0 16 16"
                            fill="none"
                        >
                            <path
                                d="M4 6L8 10L12 6"
                                stroke="currentColor"
                                stroke-width="1.5"
                                stroke-linecap="round"
                                stroke-linejoin="round"
                            />
                        </svg>
                    </button>

                    <div v-show="!stationsCollapsed" class="accordion-content">
                        <div class="item-list">
                            <div
                                v-for="(s, idx) in stations"
                                :key="s.id"
                                class="station-card"
                            >
                                <div class="station-top">
                                    <div class="row-grip">
                                        <button
                                            class="btn-icon"
                                            title="Move up"
                                            @click="moveItem(stations, idx, -1)"
                                            :disabled="idx === 0"
                                        >
                                            ↑
                                        </button>
                                        <button
                                            class="btn-icon"
                                            title="Move down"
                                            @click="moveItem(stations, idx, 1)"
                                            :disabled="
                                                idx === stations.length - 1
                                            "
                                        >
                                            ↓
                                        </button>
                                    </div>
                                    <input
                                        v-model="s.name"
                                        class="input-row input-station"
                                        placeholder="Station name"
                                    />
                                    <button
                                        class="btn-icon btn-delete"
                                        title="Delete station"
                                        @click="deleteStation(s.id)"
                                    >
                                        ×
                                    </button>
                                </div>
                                <div class="station-meta">
                                    <span class="coord-badge"
                                        >x: {{ s.x }} y: {{ s.y }}</span
                                    >
                                </div>
                                <div class="track-toggles">
                                    <label
                                        v-for="line in lineDefinitions"
                                        :key="line.id"
                                        class="toggle-pill"
                                        :style="{
                                            borderColor: s.lines.includes(
                                                line.id,
                                            )
                                                ? line.color
                                                : '#e2e8f0',
                                            background: s.lines.includes(
                                                line.id,
                                            )
                                                ? line.color + '14'
                                                : 'transparent',
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
                        </div>
                        <button
                            class="btn-add-full"
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
                                    title: '',
                                    sections: [],
                                })
                            "
                        >
                            <span class="btn-add-icon">+</span>
                            <span>Add Station</span>
                        </button>
                    </div>
                </div>
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
                                onClick: () => selectStation(item.station.id),
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
                                onClick: () => selectStation(item.station.id),
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

            <div class="track-legend">
                <div
                    v-for="line in lineDefinitions"
                    :key="line.id"
                    class="legend-item"
                >
                    <span
                        class="legend-dot"
                        :style="{ backgroundColor: line.color }"
                    ></span>
                    <span class="legend-label">{{ line.name }}</span>
                </div>
            </div>

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

        <!-- Right Sidebar -->
        <aside
            v-if="!rightSidebarCollapsed"
            class="right-sidebar"
            :style="{ width: rightSidebarWidth + 'px' }"
        >
            <div class="resize-handle" @mousedown="startResizeRight"></div>
            <div class="right-content">
                <div class="right-header">
                    <h3>{{ selectedStation.name }}</h3>
                    <button
                        class="btn-icon btn-delete"
                        title="Close panel"
                        @click="closeRightSidebar"
                    >
                        ×
                    </button>
                </div>

                <div v-if="selectedStation" class="detail-form">
                    <!-- Dynamic Sections -->
                    <div
                        v-for="(sec, idx) in selectedStation.sections"
                        :key="sec.id"
                        class="section-card"
                    >
                        <div class="section-header-row">
                            <input
                                v-model="sec.title"
                                class="section-title-input"
                                placeholder="Section title"
                            />
                            <div class="section-actions">
                                <button
                                    class="btn-icon"
                                    title="Move up"
                                    @click="moveSection(idx, -1)"
                                    :disabled="idx === 0"
                                >
                                    ↑
                                </button>
                                <button
                                    class="btn-icon"
                                    title="Move down"
                                    @click="moveSection(idx, 1)"
                                    :disabled="
                                        idx ===
                                        selectedStation.sections.length - 1
                                    "
                                >
                                    ↓
                                </button>
                                <button
                                    class="btn-icon btn-delete"
                                    title="Delete section"
                                    @click="deleteSection(sec.id)"
                                >
                                    ×
                                </button>
                                <button
                                    class="btn-icon"
                                    title="Collapse"
                                    @click="sec.collapsed = !sec.collapsed"
                                >
                                    {{ sec.collapsed ? "▸" : "▾" }}
                                </button>
                            </div>
                        </div>

                        <div v-show="!sec.collapsed" class="section-body">
                            <TipTapToolbar
                                :editor="getOrCreateEditor(sec.id, sec.content)"
                            />
                            <EditorContent
                                :editor="getOrCreateEditor(sec.id, sec.content)"
                                class="tiptap-editor"
                            />
                        </div>
                    </div>

                    <button class="btn-add-full" @click="addSection">
                        <span class="btn-add-icon">+</span>
                        <span>Add Section</span>
                    </button>
                </div>

                <div v-else class="empty-state">
                    Select a station on the map.
                </div>
            </div>
        </aside>
    </div>
</template>

<script>
// TipTap Toolbar component (inline for single-file portability)
import { h } from "vue";

const TipTapToolbar = {
    props: ["editor"],
    setup(props) {
        return () => {
            const e = props.editor;
            if (!e) return null;
            return h(
                "div",
                { class: "tiptap-toolbar" },
                [
                    { cmd: "toggleBold", label: "B", title: "Bold" },
                    { cmd: "toggleItalic", label: "I", title: "Italic" },
                    { cmd: "toggleStrike", label: "S", title: "Strike" },
                    {
                        cmd: "toggleBulletList",
                        label: "•",
                        title: "Bullet list",
                    },
                    {
                        cmd: "toggleOrderedList",
                        label: "1.",
                        title: "Ordered list",
                    },
                    { cmd: "toggleBlockquote", label: "”", title: "Quote" },
                    { cmd: "toggleCode", label: "</>", title: "Code" },
                    { cmd: "setHorizontalRule", label: "—", title: "Divider" },
                ].map((btn) =>
                    h(
                        "button",
                        {
                            class: [
                                "tool-btn",
                                {
                                    active:
                                        e.isActive &&
                                        e.isActive(
                                            btn.cmd
                                                .replace("toggle", "")
                                                .toLowerCase()
                                                .replace(
                                                    "bulletlist",
                                                    "bulletList",
                                                )
                                                .replace(
                                                    "orderedlist",
                                                    "orderedList",
                                                )
                                                .replace(
                                                    "blockquote",
                                                    "blockquote",
                                                ),
                                        ),
                                },
                            ],
                            onClick: () => e.chain().focus()[btn.cmd]().run(),
                            title: btn.title,
                        },
                        btn.label,
                    ),
                ),
            );
        };
    },
};
</script>

<style scoped>
.layout {
    display: flex;
    height: 100vh;
    background: #f1f5f9;
    font-family:
        "Inter",
        -apple-system,
        BlinkMacSystemFont,
        "Segoe UI",
        Roboto,
        sans-serif;
    overflow: hidden;
}

/* ---------- LEFT SIDEBAR ---------- */
.sidebar {
    width: 280px;
    background: #ffffff;
    border-right: 1px solid #e2e8f0;
    display: flex;
    flex-direction: column;
    overflow: hidden;
    flex-shrink: 0;
}

.app-header {
    padding: 8px 10px 6px;
    border-bottom: 1px solid #f1f5f9;
    flex-shrink: 0;
}

.app-title h1 {
    font-size: 14px;
    font-weight: 800;
    color: #0f172a;
    margin: 0 0 6px;
    letter-spacing: -0.02em;
    display: flex;
    align-items: center;
    gap: 6px;
}

.badge {
    font-size: 9px;
    font-weight: 700;
    background: #0f172a;
    color: #fff;
    padding: 1px 4px;
    border-radius: 3px;
    letter-spacing: 0.02em;
}

.app-actions {
    display: flex;
    gap: 6px;
}

.btn-toolbar {
    flex: 1;
    font-size: 11px;
    font-weight: 600;
    padding: 4px 0;
    border: 1px solid #e2e8f0;
    background: #f8fafc;
    color: #334155;
    border-radius: 5px;
    cursor: pointer;
    transition: all 0.12s ease;
}

.btn-toolbar:hover {
    background: #f1f5f9;
    border-color: #cbd5e1;
    color: #0f172a;
}

.sidebar-body {
    flex: 1;
    overflow-y: auto;
    padding: 8px;
    display: flex;
    flex-direction: column;
    gap: 8px;
}

.sidebar-body::-webkit-scrollbar {
    width: 5px;
}
.sidebar-body::-webkit-scrollbar-track {
    background: transparent;
}
.sidebar-body::-webkit-scrollbar-thumb {
    background: #cbd5e1;
    border-radius: 3px;
}

.accordion {
    background: #ffffff;
    border: 1px solid #e2e8f0;
    border-radius: 6px;
    overflow: hidden;
    display: flex;
    flex-direction: column;
    box-shadow: 0 1px 2px rgba(0, 0, 0, 0.03);
}

.accordion.collapsed {
    box-shadow: 0 1px 2px rgba(0, 0, 0, 0.02);
}

.accordion-header {
    width: 100%;
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 6px 8px;
    background: #f8fafc;
    border: none;
    cursor: pointer;
    font-family: inherit;
    transition: background 0.12s ease;
}

.accordion-header:hover {
    background: #f1f5f9;
}

.accordion-title {
    display: flex;
    align-items: center;
    gap: 6px;
}

.accordion-label {
    font-size: 11px;
    font-weight: 700;
    color: #0f172a;
    text-transform: uppercase;
    letter-spacing: 0.04em;
}

.count-badge {
    font-size: 10px;
    font-weight: 700;
    color: #64748b;
    background: #e2e8f0;
    padding: 1px 6px;
    border-radius: 999px;
    min-width: 18px;
    text-align: center;
}

.accordion-chevron {
    color: #64748b;
    transition: transform 0.2s ease;
}

.collapsed .accordion-chevron {
    transform: rotate(-90deg);
}

.accordion-content {
    padding: 6px;
    display: flex;
    flex-direction: column;
    gap: 6px;
}

.item-list {
    display: flex;
    flex-direction: column;
    gap: 4px;
    max-height: 240px;
    overflow-y: auto;
    padding-right: 2px;
}

.item-list::-webkit-scrollbar {
    width: 4px;
}
.item-list::-webkit-scrollbar-thumb {
    background: #cbd5e1;
    border-radius: 2px;
}

.track-row {
    display: flex;
    align-items: center;
    gap: 5px;
    padding: 4px 5px;
    background: #f8fafc;
    border: 1px solid #f1f5f9;
    border-radius: 5px;
    transition: border-color 0.12s ease;
}

.track-row:hover {
    border-color: #e2e8f0;
}

.color-swatch {
    width: 14px;
    height: 14px;
    border: none;
    border-radius: 50%;
    cursor: pointer;
    padding: 0;
    flex-shrink: 0;
    background: none;
    -webkit-appearance: none;
    appearance: none;
    overflow: hidden;
}

.color-swatch::-webkit-color-swatch-wrapper {
    padding: 0;
}

.color-swatch::-webkit-color-swatch {
    border: none;
    border-radius: 50%;
}

.input-row {
    flex: 1;
    border: none;
    background: transparent;
    font-size: 12px;
    font-weight: 600;
    color: #1e293b;
    outline: none;
    font-family: inherit;
    min-width: 0;
}

.input-row::placeholder {
    color: #94a3b8;
    font-weight: 500;
}

.station-card {
    padding: 5px 6px;
    background: #ffffff;
    border: 1px solid #e2e8f0;
    border-radius: 5px;
    box-shadow: 0 1px 2px rgba(0, 0, 0, 0.02);
    display: flex;
    flex-direction: column;
    gap: 4px;
}

.station-top {
    display: flex;
    align-items: center;
    gap: 5px;
}

.input-station {
    font-weight: 700;
    font-size: 12px;
}

.station-meta {
    display: flex;
    align-items: center;
    padding-left: 36px;
}

.coord-badge {
    font-size: 9px;
    font-weight: 600;
    color: #94a3b8;
    background: #f1f5f9;
    padding: 1px 5px;
    border-radius: 3px;
    letter-spacing: 0.02em;
}

.row-grip {
    display: flex;
    flex-direction: column;
    gap: 1px;
    flex-shrink: 0;
}

.btn-icon {
    width: 16px;
    height: 16px;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 3px;
    border: none;
    background: transparent;
    color: #94a3b8;
    cursor: pointer;
    font-size: 9px;
    font-weight: 700;
    line-height: 1;
    padding: 0;
    transition: all 0.1s ease;
    font-family: inherit;
}

.btn-icon:hover:not(:disabled) {
    background: #e2e8f0;
    color: #475569;
}

.btn-icon:disabled {
    opacity: 0.25;
    cursor: not-allowed;
}

.btn-delete {
    font-size: 14px;
    font-weight: 400;
    line-height: 1;
}

.btn-delete:hover:not(:disabled) {
    background: #fef2f2;
    color: #ef4444;
}

.track-toggles {
    display: flex;
    flex-wrap: wrap;
    gap: 4px;
    padding-left: 36px;
}

.toggle-pill {
    border: 1px solid #e2e8f0;
    border-radius: 4px;
    font-size: 9px;
    font-weight: 800;
    cursor: pointer;
    width: 20px;
    height: 20px;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: all 0.1s ease;
    user-select: none;
}

.toggle-pill:hover {
    transform: translateY(-1px);
}

.toggle-pill input {
    display: none;
}

.btn-add-full {
    width: 100%;
    padding: 5px;
    border: 1px dashed #cbd5e1;
    background: #f8fafc;
    color: #475569;
    border-radius: 5px;
    font-size: 11px;
    font-weight: 700;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 5px;
    transition: all 0.12s ease;
    font-family: inherit;
}

.btn-add-full:hover {
    border-color: #94a3b8;
    background: #f1f5f9;
    color: #0f172a;
}

.btn-add-icon {
    font-size: 14px;
    font-weight: 400;
    line-height: 1;
}

/* ---------- MAP ---------- */
.map-container {
    flex: 1;
    position: relative;
    cursor: crosshair;
    background: #fcfcfc;
    min-width: 0;
}

.zoom-controls {
    position: absolute;
    bottom: 16px;
    left: 16px;
    display: flex;
    gap: 6px;
    z-index: 10;
}

.zoom-btn {
    width: 30px;
    height: 30px;
    border-radius: 6px;
    border: 1px solid #e2e8f0;
    background: #ffffff;
    color: #334155;
    font-size: 16px;
    font-weight: 600;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    box-shadow: 0 1px 4px rgba(0, 0, 0, 0.06);
    transition: all 0.12s ease;
    font-family: inherit;
}

.zoom-btn:hover {
    background: #f8fafc;
    border-color: #cbd5e1;
}

/* ---------- RIGHT SIDEBAR ---------- */
.right-sidebar {
    background: #ffffff;
    border-left: 1px solid #e2e8f0;
    display: flex;
    flex-direction: column;
    position: relative;
    flex-shrink: 0;
    overflow: hidden;
}

.resize-handle {
    position: absolute;
    left: 0;
    top: 0;
    bottom: 0;
    width: 4px;
    cursor: col-resize;
    background: transparent;
    z-index: 10;
    transition: background 0.15s;
}

.resize-handle:hover {
    background: #3b82f6;
}

.right-content {
    flex: 1;
    overflow-y: auto;
    padding: 8px;
    display: flex;
    flex-direction: column;
    gap: 8px;
}

.right-content::-webkit-scrollbar {
    width: 5px;
}
.right-content::-webkit-scrollbar-thumb {
    background: #cbd5e1;
    border-radius: 3px;
}

.right-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding-bottom: 6px;
    border-bottom: 1px solid #f1f5f9;
}

.right-header h3 {
    font-size: 11px;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.04em;
    color: #0f172a;
    margin: 0;
}

.empty-state {
    font-size: 12px;
    color: #94a3b8;
    text-align: center;
    padding: 20px 0;
}

.detail-form {
    display: flex;
    flex-direction: column;
    gap: 8px;
}

.detail-label {
    font-size: 10px;
    font-weight: 700;
    color: #64748b;
    text-transform: uppercase;
    letter-spacing: 0.04em;
}

.detail-input {
    padding: 4px 6px;
    border: 1px solid #e2e8f0;
    border-radius: 4px;
    font-size: 12px;
    font-weight: 600;
    color: #1e293b;
    outline: none;
    font-family: inherit;
    background: #fff;
}

.detail-input:focus {
    border-color: #3b82f6;
}

/* ---------- SECTION CARDS ---------- */
.section-card {
    border: 1px solid #e2e8f0;
    border-radius: 6px;
    overflow: hidden;
    background: #ffffff;
}

.section-header-row {
    display: flex;
    align-items: center;
    gap: 4px;
    padding: 4px 6px;
    background: #f8fafc;
    border-bottom: 1px solid #f1f5f9;
}

.section-title-input {
    flex: 1;
    border: none;
    background: transparent;
    font-size: 12px;
    font-weight: 700;
    color: #1e293b;
    outline: none;
    font-family: inherit;
    min-width: 0;
}

.section-title-input::placeholder {
    color: #94a3b8;
}

.section-actions {
    display: flex;
    gap: 1px;
    flex-shrink: 0;
}

.section-body {
    padding: 6px;
    display: flex;
    flex-direction: column;
    gap: 4px;
}

/* ---------- TIPTAP ---------- */
.tiptap-toolbar {
    display: flex;
    gap: 2px;
    padding: 3px;
    background: #f8fafc;
    border: 1px solid #e2e8f0;
    border-radius: 4px;
    flex-shrink: 0;
    flex-wrap: wrap;
}

.tool-btn {
    width: 24px;
    height: 24px;
    border: none;
    background: transparent;
    border-radius: 3px;
    cursor: pointer;
    font-size: 11px;
    font-weight: 700;
    color: #475569;
    display: flex;
    align-items: center;
    justify-content: center;
    font-family: inherit;
    line-height: 1;
    padding: 0;
    transition: all 0.1s;
}

.tool-btn:hover {
    background: #e2e8f0;
}

.tool-btn.active {
    background: #3b82f6;
    color: #fff;
}

.tiptap-editor {
    border: 1px solid #e2e8f0;
    border-radius: 4px;
    min-height: 80px;
    max-height: 200px;
    overflow-y: auto;
    padding: 6px;
    font-size: 12px;
    line-height: 1.5;
    color: #334155;
    background: #fff;
}

.tiptap-editor :deep(.ProseMirror) {
    outline: none;
    min-height: 60px;
}

.tiptap-editor :deep(.ProseMirror p) {
    margin: 2px 0;
}

.tiptap-editor :deep(.ProseMirror ul),
.tiptap-editor :deep(.ProseMirror ol) {
    margin: 4px 0;
    padding-left: 18px;
}

.tiptap-editor :deep(.ProseMirror li) {
    margin: 2px 0;
}

.tiptap-editor :deep(.ProseMirror blockquote) {
    margin: 4px 0;
    padding-left: 8px;
    border-left: 2px solid #e2e8f0;
    color: #64748b;
}

.tiptap-editor :deep(.ProseMirror code) {
    background: #f1f5f9;
    padding: 1px 3px;
    border-radius: 3px;
    font-size: 11px;
    font-family:
        ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;
}

.tiptap-editor :deep(.ProseMirror hr) {
    border: none;
    border-top: 1px solid #e2e8f0;
    margin: 8px 0;
}

/* ---------- TRACK LEGEND ---------- */
.track-legend {
    position: absolute;
    bottom: 16px;
    right: 16px;
    background: #ffffff;
    border: 1px solid #e2e8f0;
    border-radius: 8px;
    padding: 8px 12px;
    display: flex;
    flex-direction: column;
    gap: 6px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
    z-index: 10;
    min-width: 120px;
}

.legend-item {
    display: flex;
    align-items: center;
    gap: 8px;
}

.legend-dot {
    width: 10px;
    height: 10px;
    border-radius: 50%;
    flex-shrink: 0;
}

.legend-label {
    font-size: 11px;
    font-weight: 600;
    color: #334155;
    white-space: nowrap;
}
</style>
