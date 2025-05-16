<script setup>
import { onKeyDown, onKeyUp, useStorage } from "@vueuse/core";
import { Chord, Note, Range, Midi } from "tonal";
import { reactive, computed, ref, onMounted } from "vue";
import {
  noteColor,
  notes,
  rotateArray,
  playNote,
  stopNote,
  getCircleCoord,
  useSoundFont,
  useMidi,
} from "use-chromatone";
import { colord } from "colord";
import { useGesture } from "@vueuse/gesture";

import SvgRing from "./SvgRing.vue";

const {
  getSoundfontNames,
  inst,
  cached,
  loaded,
  instrument,
  synthEnabled,
  active,
  volume,
} = useSoundFont();

const { midi, activeChroma, guessChords } = useMidi();

const keyToNote = {
  a: "C",
  s: "G",
  d: "D",
  f: "A",
  g: "E",
  h: "B",
  j: "F#",
  k: "C#",
  l: "G#",
  z: "Cm",
  x: "Gm",
  c: "Dm",
  v: "Am",
  b: "Em",
  n: "Bm",
  m: "F#m",
};

const rotatedNotes = computed(() => {
  const notes = Object.values(keyToNote);
  const rotation = tonic.value;
  return notes.map((note, i) => {
    const isMinor = note.endsWith("m");
    const baseNote = isMinor ? note.slice(0, -1) : note;
    const index = notes.findIndex((n) => n === baseNote);
    const rotatedIndex = (index + rotation) % 12;
    const rotatedNote = notes[rotatedIndex];
    return isMinor ? rotatedNote + "m" : rotatedNote;
  });
});

const state = reactive({
  seventh: false,
  main: true,
  pressed: false,
  chordsMode: false,
  soundEnabled: true,
  activeKeys: new Set(),
});

const browserSound = reactive({
  enabled: true,
  volume: 0.3,
});

onKeyDown((ev) => {
  if (ev.key === "ArrowLeft") {
    tonic.value = (tonic.value - 1 + 12) % 12;
    return;
  }
  if (ev.key === "ArrowRight") {
    tonic.value = (tonic.value + 1) % 12;
    return;
  }

  if (!state.chordsMode) return;

  const key = ev.key.toLowerCase();
  const index = Object.keys(keyToNote).indexOf(key);
  if (index !== -1 && !state.activeKeys.has(key)) {
    state.activeKeys.add(key);
    const note = rotatedNotes.value[index];
    const isMinor = note.endsWith("m");
    const baseNote = isMinor ? note.slice(0, -1) : note;
    playChord(baseNote, isMinor ? "minor" : "major");
  }
});

onKeyUp((ev) => {
  if (!state.chordsMode) return;

  const key = ev.key.toLowerCase();
  const index = Object.keys(keyToNote).indexOf(key);
  if (index !== -1 && state.activeKeys.has(key)) {
    state.activeKeys.delete(key);
    const note = rotatedNotes.value[index];
    const isMinor = note.endsWith("m");
    const baseNote = isMinor ? note.slice(0, -1) : note;
    stopChord(baseNote, isMinor ? "minor" : "major");
  }
});

onKeyDown("Shift", (ev) => {
  state.seventh = !state.seventh;
});
onKeyUp("Shift", (ev) => {
  state.seventh = !state.seventh;
});
onKeyDown("Alt", (ev) => {
  state.main = !state.main;
});
onKeyUp("Alt", (ev) => {
  state.main = !state.main;
});
onKeyDown("c", (ev) => {
  if (ev.ctrlKey || ev.metaKey) {
    state.chordsMode = !state.chordsMode;
  }
});

const numFifths = [0, 7, 2, 9, 4, 11, 6, 1, 8, 3, 10, 5];
const allNotes = notes.map((n, i) => ({ name: n, pitch: i }));
const minors = numFifths.map((n) => allNotes[n]);
const majors = rotateArray(numFifths, -3).map((n) => allNotes[n]);

const scales = { minor: minors, major: majors };

const tonic = useStorage("tonic", 0);
const scaleType = useStorage("scale-type", "major");

const steps = {
  minor: [
    ["VI", "III", "VII"],
    ["iv", "i", "v"],
  ],
  major: [
    ["IV", "I", "V"],
    ["ii", "vi", "iii"],
  ],
};

const chordShapes = {
  minor: [0, 3, 7, 12],
  major: [0, 4, 7, 12],
  minor7: [0, 3, 7, 10],
  major7: [0, 4, 7, 11],
};

function getRadius(qual) {
  return qual == "minor" ? 1 : 0;
}

function getChordType(qual) {
  return qual == "minor"
    ? state.seventh
      ? "m7"
      : "m"
    : state.seventh
    ? "M7"
    : "";
}

function getChordNotes(note, qual = "major", inv) {
  const type = getChordType(qual);
  const chord = Chord.get(note + type);

  if (inv !== undefined) {
    const result = Range.numeric([1 + inv, 4 + inv]).map(
      Chord.degrees(type || "major", note + `3`)
    );
    return result;
  } else {
    return Note.names(chord.notes.map((n) => Note.simplify(n) + 4));
  }
}

function playChord(note, qual = "major", inv) {
  const notes = getChordNotes(note, qual, inv);

  // Gestion MIDI
  if (midi.out && midi.port) {
    notes.forEach((note) => {
      const midiNote = Note.midi(note);
      if (midiNote) {
        midi.port.send([0x90 + (midi.channel - 1), midiNote, 100]);
      }
    });
    console.log("Playing MIDI:", { notes, channel: midi.channel });
  }

  // Gestion son navigateur
  if (state.soundEnabled) {
    playNote(notes);
  }
}

function stopChord(note, qual = "major", inv) {
  const notes = getChordNotes(note, qual, inv);

  // Gestion MIDI
  if (midi.out && midi.port) {
    notes.forEach((note) => {
      const midiNote = Note.midi(note);
      if (midiNote) {
        midi.port.send([0x80 + (midi.channel - 1), midiNote, 0]);
      }
    });
  }

  // Gestion son navigateur
  if (state.soundEnabled) {
    stopNote(notes);
  }
}

const capitalize = (s) => s && s[0].toUpperCase() + s.slice(1);

const svg = ref();
const touchPoints = new Map();
const currentNote = ref(null);

useGesture(
  {
    onTouchstart: handleTouchStart,
    onTouchmove: handleTouchMove,
    onTouchend: handleTouchEnd,
    onTouchcancel: handleTouchEnd,
    onDrag: ({ event, first, last, active }) => {
      event.preventDefault();
      const element = document.elementFromPoint(event.clientX, event.clientY);
      if (!element) return;

      const quadroElement = element.closest(".quadro");
      const circleElement = !quadroElement && element.closest(".circle-note");

      if (!quadroElement && !circleElement) return;

      const el = quadroElement || circleElement;
      const note = el.dataset.note;
      const qual = el.dataset.qual;
      const inv = parseInt(el.dataset.inv || "0");

      if (!note) return;

      if (first) {
        playChord(note, qual, inv);
        currentNote.value = { note, qual, inv };
      } else if (last) {
        if (currentNote.value) {
          stopChord(
            currentNote.value.note,
            currentNote.value.qual,
            currentNote.value.inv
          );
          currentNote.value = null;
        }
      } else if (
        active &&
        (currentNote.value?.note !== note || currentNote.value?.inv !== inv)
      ) {
        if (currentNote.value) {
          stopChord(
            currentNote.value.note,
            currentNote.value.qual,
            currentNote.value.inv
          );
        }
        playChord(note, qual, inv);
        currentNote.value = { note, qual, inv };
      }
    },
    onDragEnd: () => {
      if (currentNote.value) {
        stopChord(
          currentNote.value.note,
          currentNote.value.qual,
          currentNote.value.inv
        );
        currentNote.value = null;
      }
    },
  },
  {
    domTarget: svg,
    eventOptions: { passive: false },
    triggerAllEvents: true,
    dragDelay: 0,
  }
);

function handleTouchStart({ event }) {
  event.preventDefault();
  Array.from(event.changedTouches).forEach((touch) => {
    const element = document.elementFromPoint(touch.clientX, touch.clientY);
    if (!element) return;

    const quadroElement = element.closest(".quadro");
    const circleElement = !quadroElement && element.closest(".circle-note");

    if (!quadroElement && !circleElement) return;

    const el = quadroElement || circleElement;
    const note = el.dataset.note;
    const qual = el.dataset.qual;
    const inv = parseInt(el.dataset.inv || "0");

    if (!note) return;

    touchPoints.set(touch.identifier, { note, qual, inv });
    playChord(note, qual, inv);
  });
}

function handleTouchMove({ event }) {
  event.preventDefault();
  Array.from(event.changedTouches).forEach((touch) => {
    const oldData = touchPoints.get(touch.identifier);
    const element = document.elementFromPoint(touch.clientX, touch.clientY);
    if (!element) {
      if (oldData) {
        stopChord(oldData.note, oldData.qual, oldData.inv);
        touchPoints.delete(touch.identifier);
      }
      return;
    }

    const quadroElement = element.closest(".quadro");
    const circleElement = !quadroElement && element.closest(".circle-note");

    if (!quadroElement && !circleElement) {
      if (oldData) {
        stopChord(oldData.note, oldData.qual, oldData.inv);
        touchPoints.delete(touch.identifier);
      }
      return;
    }

    const el = quadroElement || circleElement;
    const note = el.dataset.note;
    const qual = el.dataset.qual;
    const inv = parseInt(el.dataset.inv || "0");

    if (!note) return;

    if (!oldData || oldData.note !== note || oldData.inv !== inv) {
      if (oldData) {
        stopChord(oldData.note, oldData.qual, oldData.inv);
      }
      touchPoints.set(touch.identifier, { note, qual, inv });
      playChord(note, qual, inv);
    }
  });
}

function handleTouchEnd({ event }) {
  event.preventDefault();
  Array.from(event.changedTouches).forEach((touch) => {
    const data = touchPoints.get(touch.identifier);
    if (data) {
      stopChord(data.note, data.qual, data.inv);
      touchPoints.delete(touch.identifier);
    }
  });
}

// Configuration MIDI
onMounted(async () => {
  try {
    const midiAccess = await navigator.requestMIDIAccess();
    console.log("MIDI Access:", midiAccess);
    const outputs = Array.from(midiAccess.outputs.values());
    console.log("Available MIDI outputs:", outputs);
    if (outputs.length > 0) {
      midi.port = outputs[0];
    }
    const inputs = Array.from(midiAccess.inputs.values());
    if (inputs.length > 0) {
      midi.inPort = inputs[0];
    }
  } catch (err) {
    console.error("MIDI Access Error:", err);
  }
});

// Réinitialiser le volume par défaut
onMounted(() => {
  browserSound.volume = 0.3;
  browserSound.enabled = true;
});

// Initialisation des canaux MIDI
onMounted(() => {
  midi.inChannel = 1;
  midi.channel = 1;
});
</script>

<template lang="pug">
.fullscreen-container#screen.select-none.touch-manipulation.h-full.max-h-screen
  .flex.fixed.items-center.top-4.left-4.right-4.gap-2
    button.text-2xl(
      v-tooltip="`Toggle sampler synth `"
      :style="{ opacity: synthEnabled ? 1 : 0.5 }" @click="synthEnabled = !synthEnabled") {{ loaded ? '⌽' : '✲' }}
    select.rounded-lg.text-sm.min-w-8.p-2.bg-dark-500(
      v-tooltip="`Select sampler synth sound`"
      v-model="instrument"
      @change="$event.target.blur()")
      option( v-for="name in getSoundfontNames()" :key="name" :value="name") {{ capitalize(name.replaceAll('_', ' ')) }} {{ cached[name] ? '✔' : '' }}
    .flex-1 
    input.rounded-lg.text-sm.min-w-8.p-2.bg-dark-500(
      type="range"
      min="0"
      max="1"
      step="0.1"
      v-model="volume"
      v-tooltip="`Volume: ${Math.round(volume * 100)}%`"
    )
    button.rounded-lg.text-sm.px-3.py-2.bg-dark-500.flex.items-center.gap-2(
      v-tooltip="`Toggle keyboard mode (Ctrl/Cmd + C)`"
      :class="{ 'bg-primary-500': state.chordsMode }"
      @click="state.chordsMode = !state.chordsMode")
      .i-mdi-keyboard.text-xl
      span.text-sm {{ state.chordsMode ? 'Key Chords On' : 'Key Chords Off' }}
  .flex.fixed.items-center.bottom-4.left-4.right-4
    .flex.items-center.gap-2.p-2.rounded-lg
      .flex.items-center.gap-2
        button.rounded-lg.text-sm.px-3.py-2.bg-dark-400.flex.items-center.gap-2(
          v-tooltip="`Toggle MIDI input from keyboard`"
          :class="{ 'bg-primary-500': midi.in }"
          @click="midi.in = !midi.in")
          .i-mdi-midi.text-xl
          span.text-sm {{ midi.in ? 'In On' : 'In Off' }}
        select.rounded-lg.text-sm.min-w-7.p-2.px-3.bg-dark-400(
          v-tooltip="`Select MIDI input channel`"
          v-model="midi.inChannel"
          @change="$event.target.blur()")
          option( v-for="ch in Array(16).fill(true).map((_, i) => i + 1)" :key="ch" :value="ch") Ch {{ ch }}
    .flex-1
    .flex.items-center.gap-2.p-2.rounded-lg
      .flex.items-center.gap-2
        button.rounded-lg.text-sm.px-3.py-2.bg-dark-400.flex.items-center.gap-2(
          v-tooltip="`Toggle MIDI output to DAW`"
          :class="{ 'bg-primary-500': midi.out }"
          @click="midi.out = !midi.out")
          .i-mdi-midi.text-xl
          span.text-sm {{ midi.out ? 'Out On' : 'Out Off' }}
        select.rounded-lg.text-sm.min-w-7.p-2.px-3.bg-dark-400(
          v-tooltip="`Select MIDI output channel`"
          v-model="midi.channel"
          @change="$event.target.blur()")
          option( v-for="ch in Array(16).fill(true).map((_, i) => i + 1)" :key="ch" :value="ch") Ch {{ ch }}
  svg#fifths.w-full.h-full(
    style="flex: 1 1 auto; touch-action:none;user-select: none; -webkit-user-select: none; -webkit-touch-callout: none;"
    version="1.1",
    baseProfile="full",
    viewBox="0 0 100 100",
    xmlns="http://www.w3.org/2000/svg",
    font-family="Commissioner, sans-serif"
    ref="svg"
    )
    g(
      fill="currentColor"
      transform="translate(50,50)"
      font-size="3px"
      text-anchor="middle",
      dominant-baseline="middle"
      font-weight="bold"
      )
      text {{ guessChords[0] }}
    g.cursor-pointer(
      transform="translate(10,90)"
      @pointerdown="state.seventh = !state.seventh"
      v-tooltip.top="'Hold SHIFT to toggle 7th chords'"
      )
      text(
        fill="currentColor"
        font-size="3px"
        text-anchor="middle",
        dominant-baseline="middle"
        y="0.3"
        :font-weight="state.seventh ? 'bold' : 'normal'"
        ) 7
      circle(
        r="3"
        fill="transparent"
        stroke="currentColor"
        :stroke-width="state.seventh ? 0.8 : 0.4"
        )
    g.cursor-pointer(
      transform="translate(90,90)"
      @pointerdown="state.main = !state.main"
      v-tooltip.top="'Hold ALT to toggle chord inversions'"
      )
      text(
        fill="currentColor"
        font-size="3px"
        text-anchor="middle",
        dominant-baseline="middle"
        y="0.3"
        :font-weight="state.main ? 'bold' : 'normal'"
        ) A
      circle(
        r="3"
        fill="transparent"
        stroke="currentColor"
        :stroke-width="state.main ? 0.8 : 0.4"
        )
    g.rings(

      )
      g.ring(
        v-for="(scale, qual) in scales"
        :key="qual"
        )
        g.around(
          v-for="(note, i) in scale", 
          :key="i",
          style="cursor:pointer"
          )
          svg-ring(
            :cx="50"
            :cy="50"
            :from="(i - 1) / 12 * 360 + 15"
            :to="(i) / 12 * 360 + 15"
            :radius="40 - 10 * getRadius(qual)"
            :thickness="10"
            :op="Math.abs(tonic - i) == 11 || Math.abs(tonic - i) % 12 <= 1 ? 0.8 : 0.1"
            :fill="Math.abs(tonic - i) == 11 || Math.abs(tonic - i) % 12 <= 1 ? noteColor(note.pitch) : noteColor(note.pitch, 2, 1)"
            )
          g.quadro(

            v-for="(deg, j) in chordShapes[qual + (state.seventh ? '7' : '')]"
            :key="j"
            :data-note="note.name"
            :data-qual="qual"
            :data-inv="j"
            )
            svg-ring.transition(
              :cx="50"
              :cy="50"
              :from="(i - 1) / 12 * 360 + 15 + 15 * (j % 2)"
              :to="(i) / 12 * 360 + 15 * (j % 2)"
              :radius="40 - 10 * getRadius(qual) - 5 * (j > 1 ? 0 : 1)"
              :thickness="5"
              :op="Math.abs(tonic - i) == 11 || Math.abs(tonic - i) % 12 <= 1 ? activeChroma[(note.pitch + deg) % 12] == 1 ? .7 : .3 : activeChroma[(note.pitch + deg) % 12] == 1 ? .2 : 0.1"
              :fill="Math.abs(tonic - i) == 11 || Math.abs(tonic - i) % 12 <= 1 ? noteColor(note.pitch + deg, 5) : noteColor(note.pitch + deg, 5, 1)"
              )
          circle.transition(
            :cx="getCircleCoord(i, 12, 42 - getRadius(qual) * 26).x",
            :cy="getCircleCoord(i, 12, 42 - getRadius(qual) * 26).y",
            :r="2"
            :fill="noteColor(note.pitch, 4, 1, 1)"
            class="opacity-20 hover-opacity-80"
            @click="tonic = i; scaleType = qual"
            )
          g.circle-note(
            v-if="state.main"
            :data-note="note.name"
            :data-qual="qual"

            )
            circle.note.opacity-80.hover-opacity-100(
              style="transition: all 300ms ease-out;transform-box: fill-box; transform-origin: center center;"
              :cx="getCircleCoord(i, 12, 35 - getRadius(qual) * 12).x",
              :cy="getCircleCoord(i, 12, 35 - getRadius(qual) * 12).y",
              r="5",
              :fill="Math.abs(tonic - i) == 11 || Math.abs(tonic - i) % 12 <= 1 ? noteColor(note.pitch, 4) : noteColor(note.pitch, 5, 1, 0.5)",
            )
          text.pointer-events-none(
            style="user-select:none;transition:all 300ms ease"
            :fill="colord(noteColor(note.pitch, 4)).isDark() ? 'white' : 'white'"
            font-size="3px"
            text-anchor="middle",
            dominant-baseline="middle"
            :x="getCircleCoord(i, 12, 35 - getRadius(qual) * 12).x",
            :y="getCircleCoord(i, 12, 35 - getRadius(qual) * 12).y + 0.5",
          ) {{ note.name }}{{ getChordType(qual) }}

    g.transition-all.duration-300.ease-out(
      ref="selector"
      transform-origin="50 50"
      :style="{ transform: `rotate(${tonic / 12 * 360}deg)` }"
      )
      svg-ring(
        :cx="50"
        :cy="50"
        :from="(-2) / 12 * 360 + 15"
        :to="(+ 1) / 12 * 360 + 15"
        :radius="44.5"
        :thickness="31"
        :sWidth=".5"
        stroke="#fff3"
        fill="none"
        )
      circle.transition-all.duration-300.cursor-pointer(
        v-if="scaleType != 'minor'"
        :cx="50"
        :cy="8"
        :r="2"
        :fill="noteColor(majors[tonic].pitch, 4)"
      )
      circle.transition-all.duration-300.cursor-pointer(
        v-if="scaleType != 'major'"
        :cx="50"
        :cy="34"
        :r="2"
        :fill="noteColor(minors[tonic].pitch, 4)"
      )
      g(
        v-for="(level, idx) in steps[scaleType]" 
        :key="idx")
        text.pointer-events-none(
          v-for="(step, n) in level"
          :key="step"
          style="user-select:none;transition:all 300ms ease;"
          fill="currentColor"
          font-size="2.5px"
          text-anchor="middle",
          dominant-baseline="middle"
          :x="getCircleCoord(n - 1, 12, 42 - idx * 26).x",
          :y="getCircleCoord(n - 1, 12, 42 - idx * 26).y + 0.25",
          :style="{ transform: `rotate(${-(tonic) * 30}deg)`, transformOrigin: `${getCircleCoord(n - 1, 12, 42 - idx * 26).x}px ${getCircleCoord(n - 1, 12, 42 - idx * 26).y}px` }"
        ) {{ step }}
</template>

<style lang="postcss" scoped>
g.active path,
path:hover {
  @apply opacity-100;
}

svg {
  touch-action: none;
  -webkit-touch-callout: none;
  -webkit-user-select: none;
  user-select: none;
}

.circle-note {
  pointer-events: none;
}

.quadro {
  touch-action: none;
  pointer-events: all;
}

select {
  appearance: none;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' fill='none' viewBox='0 0 24 24' stroke='white'%3E%3Cpath stroke-linecap='round' stroke-linejoin='round' stroke-width='2' d='M19 9l-7 7-7-7'%3E%3C/path%3E%3C/svg%3E");
  background-position: right 0.5rem center;
  background-repeat: no-repeat;
  background-size: 1em 1em;
  padding-right: 2rem;
}
</style>
