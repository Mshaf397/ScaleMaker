<script setup lang="ts">
import { computed, onMounted, onUnmounted, reactive, watch } from 'vue'
import { RouterLink, RouterView, useRouter } from 'vue-router'
import { DEFAULT_NUMBER_OF_COMPONENTS } from '@/constants'
import { ScaleWorkshopOneData } from '@/scale-workshop-one'
import type { Input, Output } from 'webmidi'
import { MidiIn, midiKeyInfo, MidiOut, type NoteOff } from 'xen-midi'
import { Keyboard, type CoordinateKeyboardEvent, COORDS_BY_CODE } from 'isomorphic-qwerty'
import { decodeQuery } from '@/url-encode'
import { annotateColors } from '@/utils'
import { version } from '../package.json'
import { useAudioStore } from '@/stores/audio'
import { useStateStore } from './stores/state'
import { useMidiStore } from './stores/midi'
import { useScaleStore } from './stores/scale'
import { useHarmonicEntropyStore } from '@/stores/harmonic-entropy'
import { clamp, mmod } from 'xen-dev-utils'
import { parseScaleWorkshop2Line, setNumberOfComponents } from 'sonic-weave'

// === Pinia-managed state ===
const state = useStateStore()
const scale = useScaleStore()
const midi = useMidiStore()
const audio = useAudioStore()
const entropy = useHarmonicEntropyStore()

// == URL path handling ==
/**
 * Strip away base path "/"
 */
function getPath(url: URL) {
  return url.pathname.slice(import.meta.env.BASE_URL.length)
}

const router = useRouter()

// === Tuning table highlighting ===
function tuningTableKeyOn(index: number) {
  state.heldNotes.set(index, (state.heldNotes.get(index) ?? 0) + 1)
}

function tuningTableKeyOff(index: number) {
  state.heldNotes.set(index, Math.max(0, (state.heldNotes.get(index) ?? 0) - 1))
}

// === MIDI input / output ===

const midiOut = computed(() => new MidiOut(midi.output as Output, midi.outputChannels))

function sendNoteOn(index: number, frequency: number, rawAttack: number) {
  frequency = clamp(-24000, 24000, frequency)
  if (isNaN(frequency)) {
    frequency = 0
  }
  let midiOff: NoteOff = () => {}
  if (midi.outputMode === 'pitchBend') {
    midiOff = midiOut.value.sendNoteOn(frequency, rawAttack)
  } else if (midi.output !== null) {
    midi.output.sendNoteOn(index, { channels: midi.outputChannel, rawAttack })
    midiOff = (rawRelease?: number, time?: DOMHighResTimeStamp) => {
      if (midi.output !== null) {
        midi.output.sendNoteOff(index, { channels: midi.outputChannel, rawRelease, time })
      }
    }
  }

  if (audio.synth === null || audio.virtualSynth === null) {
    return midiOff
  }

  // Trigger web audio API synth.
  const synthOff = audio.synth.noteOn(frequency, rawAttack / 127)

  // Trigger virtual synth for per-voice visualization.
  const virtualOff = audio.virtualSynth.voiceOn(frequency)

  const off = (rawRelease: number) => {
    midiOff(rawRelease)
    synthOff()
    virtualOff()
  }

  return off
}

function midiNoteOn(index: number, rawAttack?: number, channel?: number) {
  const multichannel = midi.multichannelToEquave

  // in multichannel-to-equave mode calculate an offset based on the incoming channel
  if (multichannel && channel !== undefined) {
    let offset =
      mmod(
        channel - midi.multichannelCenter + midi.multichannelEquavesDown,
        midi.multichannelNumEquaves
      ) - midi.multichannelEquavesDown
    offset = offset * scale.scale.size
    index = index + offset
  }

  if (rawAttack === undefined) {
    rawAttack = 80
  }

  if (!midi.velocityOn) {
    rawAttack = 80
  }

  // since index can go out of range in multichannel-to-equave mode, use getFrequency()
  let frequency = scale.getFrequency(index)

  // Store state to ensure consistent note off.
  const info = midiKeyInfo(index)
  const whiteMode = midi.whiteMode
  const indices = scale.whiteIndices

  if (whiteMode === 'off' || multichannel) {
    tuningTableKeyOn(index)
  } else if (whiteMode === 'simple') {
    if (info.whiteNumber === undefined) {
      frequency = NaN
    } else {
      info.whiteNumber += scale.whiteModeOffset
      frequency = scale.getFrequency(info.whiteNumber)
      tuningTableKeyOn(info.whiteNumber)
    }
  } else if (whiteMode === 'blackAverage') {
    if (info.whiteNumber === undefined) {
      info.flatOf += scale.whiteModeOffset
      info.sharpOf += scale.whiteModeOffset
      frequency = Math.sqrt(scale.getFrequency(info.flatOf) * scale.getFrequency(info.sharpOf))
      tuningTableKeyOn(info.flatOf)
      tuningTableKeyOn(info.sharpOf)
    } else {
      info.whiteNumber += scale.whiteModeOffset
      frequency = scale.getFrequency(info.whiteNumber)
      tuningTableKeyOn(info.whiteNumber)
    }
  } else if (whiteMode === 'keyColors') {
    if (indices.length) {
      if (info.whiteNumber === undefined) {
        // Use a black key if available
        index = indices[info.sharpOf] + 1
        // Eliminate duplicates
        if (index === indices[info.sharpOf + 1]) {
          frequency = NaN
        } else {
          frequency = scale.getFrequency(index)
          tuningTableKeyOn(index)
        }
      } else {
        index = indices[info.whiteNumber]
        frequency = scale.getFrequency(index)
        tuningTableKeyOn(index)
      }
    } else {
      frequency = NaN
    }
  }

  if (isNaN(frequency)) {
    // eslint-disable-next-line @typescript-eslint/no-unused-vars
    return (rawRelease?: number) => {}
  }

  const noteOff = sendNoteOn(index, frequency, rawAttack)
  return (rawRelease?: number) => {
    if (rawRelease === undefined) {
      rawRelease = 80
    }
    if (!midi.velocityOn) {
      rawRelease = 80
    }
    if (whiteMode === 'off' || whiteMode === 'keyColors' || multichannel) {
      tuningTableKeyOff(index)
    } else if (whiteMode === 'simple') {
      if (info.whiteNumber !== undefined) {
        tuningTableKeyOff(info.whiteNumber)
      }
    } else if (whiteMode === 'blackAverage') {
      if (info.whiteNumber === undefined) {
        tuningTableKeyOff(info.flatOf)
        tuningTableKeyOff(info.sharpOf)
      } else {
        tuningTableKeyOff(info.whiteNumber)
      }
    }
    noteOff(rawRelease)
  }
}

const midiInputChannels = reactive(new Set([1]))

const midiIn = new MidiIn(midiNoteOn, midiInputChannels)

const RESERVED_MESSAGES = ['noteon', 'noteoff', 'pitchbend']
watch(
  () => midi.input,
  (newValue, oldValue) => {
    if (oldValue !== null) {
      midiIn.unlisten(oldValue as Input)
    }
    if (newValue !== null) {
      midiIn.listen(newValue as Input)

      // Pass everything else through and distribute among the channels
      ;(newValue as Input).addListener('midimessage', (event) => {
        if (!RESERVED_MESSAGES.includes(event.message.type) && midi.output !== null) {
          if (event.message.isChannelMessage) {
            if (midiInputChannels.has(event.message.channel)) {
              const status = event.message.statusByte & 0b11110000
              const data = [...event.message.data]
              if (midi.outputMode === 'pitchBend') {
                for (const channel of midi.outputChannels) {
                  data[0] = status | (channel - 1)
                  midi.output.send(data)
                }
              } else {
                data[0] = status | (midi.outputChannel - 1)
                midi.output.send(data)
              }
            }
          } else {
            midi.output.send(event.message)
          }
        }
      })
    }
  }
)

// === Virtual and typing keyboard ===
function keyboardNoteOn(index: number) {
  tuningTableKeyOn(index)
  const noteOff = sendNoteOn(index, scale.getFrequency(index), 80)
  function keyOff() {
    tuningTableKeyOff(index)
    return noteOff(80)
  }
  return keyOff
}

// === Typing keyboard state ===
function windowKeydownOrUp(event: KeyboardEvent | MouseEvent) {
  // Audio context must be initialized as a response to user gesture
  setTimeout(() => audio.initialize(), 1)

  const target = event.target
  // Keep typing activated while adjusting sliders
  if (target instanceof HTMLInputElement && ['range', 'radio', 'checkbox'].includes(target.type)) {
    state.typingActive = true
    return
  }
  // Disable typing for other types of input elements
  if (
    target instanceof HTMLTextAreaElement ||
    target instanceof HTMLInputElement ||
    target instanceof HTMLSelectElement
  ) {
    state.typingActive = false
  } else {
    state.typingActive = true
  }
}

// === Handle special keys ===
function windowKeydown(event: KeyboardEvent) {
  // Currently editing the scale, bail out
  if (!state.typingActive) {
    return
  }

  if (event.ctrlKey) {
    // Allow copy & paste.
    return
  } else if (event.altKey || event.metaKey) {
    // Allow keyboard navigation out of the app.
    return
  } else if (
    [
      state.deactivationCode,
      state.equaveUpCode,
      state.equaveDownCode,
      state.degreeUpCode,
      state.degreeDownCode
    ].includes(event.code)
  ) {
    // Prevent overlapping action with configurable state.
    event.preventDefault()
  } else if (COORDS_BY_CODE.has(event.code) && COORDS_BY_CODE.get(event.code)![2] === 1) {
    // Prevent action for keys that make sound.
    event.preventDefault()
  } else if (event.key === '/') {
    // Disable browser specific features like quick find on Firefox.
    event.preventDefault()
  }

  // The key left of Digit1 releases sustained keys
  if (event.code === state.deactivationCode) {
    typingKeyboard.deactivate()
    return
  }

  // "Octave" keys
  if (event.code === state.equaveUpCode) {
    scale.equaveShift++
    return
  }
  if (event.code === state.equaveDownCode) {
    scale.equaveShift--
    return
  }

  // "Transpose" keys
  if (event.code === state.degreeUpCode) {
    scale.degreeShift++
    return
  }
  if (event.code === state.degreeDownCode) {
    scale.degreeShift--
    return
  }

  typingKeyboard.keydown(event)
}

// Keyups don't make new sounds so they can be passed through.
function windowKeyup(event: KeyboardEvent) {
  typingKeyboard.keyup(event)
}

// === Typing keyboard input ===
const typingKeyboard = new Keyboard()

function emptyKeyup() {}

function typingKeydown(event: CoordinateKeyboardEvent) {
  // Key not mapped to layers, bail out
  if (event.coordinates === undefined) {
    return emptyKeyup
  }

  const [x, y, z] = event.coordinates

  // Key not in the layer with digits and letters, bail out
  if (z !== 1) {
    return emptyKeyup
  }

  let index = scale.scale.baseMidiNote + scale.scale.size * scale.equaveShift + scale.degreeShift

  if (scale.keyboardMode === 'isomorphic') {
    index += x * scale.isomorphicHorizontal + (2 - y) * scale.isomorphicVertical
  } else {
    if (scale.qwertyMapping.has(event.code)) {
      // QWERTY mapping incorporates shifts
      index = scale.qwertyMapping.get(event.code)!
    } else {
      // No user mapping for the key, bail out
      return emptyKeyup
    }
  }

  return keyboardNoteOn(index)
}

// === Lifecycle ===
onMounted(async () => {
  window.addEventListener('keyup', windowKeyup)
  window.addEventListener('keydown', windowKeydownOrUp)
  window.addEventListener('keyup', windowKeydownOrUp)
  window.addEventListener('mousedown', windowKeydownOrUp)
  window.addEventListener('keydown', windowKeydown)
  window.addEventListener('touchstart', audio.initialize)
  typingKeyboard.addKeydownListener(typingKeydown)

  const url = new URL(window.location.href)
  const query = url.searchParams

  // This is overridden when scale data is evaluated, but some corner cases need to be covered.
  setNumberOfComponents(DEFAULT_NUMBER_OF_COMPONENTS)

  // Special handling for the empty app state so that
  // the browser's back button can undo to the clean state.
  if (![...query.keys()].length) {
    await router.push({ path: getPath(url), query: { version } })
  } else if (!query.has('version')) {
    // Scale Workshop 1 compatibility
    try {
      const scaleWorkshopOneData = new ScaleWorkshopOneData()
      audio.initialize()

      scale.name = scaleWorkshopOneData.name
      scale.userBaseFrequency = scaleWorkshopOneData.freq
      scale.autoFrequency = false
      scale.baseMidiNote = scaleWorkshopOneData.midi
      scale.isomorphicHorizontal = scaleWorkshopOneData.horizontal
      scale.isomorphicVertical = scaleWorkshopOneData.vertical

      if (scaleWorkshopOneData.data !== undefined) {
        const colors = scaleWorkshopOneData.colors ?? ''
        const intervals = scaleWorkshopOneData.parseTuningData()
        // Convert to raw text
        const sourceLines = intervals.map((i) => i.toString())
        annotateColors(sourceLines, colors.split(' '))
        scale.sourceText = sourceLines.join('\n')
        scale.computeScale()
        scale.history.truncate()
      }

      audio.waveform = scaleWorkshopOneData.waveform || 'semisine'
      audio.attackTime = scaleWorkshopOneData.attackTime
      audio.decayTime = scaleWorkshopOneData.decayTime
      audio.sustainLevel = scaleWorkshopOneData.sustainLevel
      audio.releaseTime = scaleWorkshopOneData.releaseTime

      // Replace query with version 3.
      await router.push({ path: getPath(url), query: { version } })
    } catch (error) {
      console.error('Error parsing version 1 URL', error)
    }
  } else if (query.get('version')!.startsWith('2.')) {
    // Scale Workshop 2 compatibility
    try {
      const decodedState = decodeQuery(query)
      audio.initialize()

      let pianoMode: 'Asdf' | 'QweZxc' = 'Asdf'
      if (decodedState.pianoMode === 'QweZxc0' || decodedState.pianoMode === 'QweZxc1') {
        pianoMode = 'QweZxc'
      }

      scale.name = decodedState.scaleName
      scale.userBaseFrequency = decodedState.baseFrequency
      scale.autoFrequency = false
      scale.baseMidiNote = decodedState.baseMidiNote
      scale.isomorphicHorizontal = decodedState.isomorphicHorizontal
      scale.isomorphicVertical = decodedState.isomorphicVertical
      scale.keyboardMode = decodedState.keyboardMode
      scale.pianoMode = pianoMode
      scale.equaveShift = decodedState.equaveShift
      scale.degreeShift = decodedState.degreeShift
      audio.waveform = decodedState.waveform
      audio.attackTime = decodedState.attackTime
      audio.decayTime = decodedState.decayTime
      audio.sustainLevel = decodedState.sustainLevel
      audio.releaseTime = decodedState.releaseTime
      audio.pingPongDelayTime = decodedState.pingPongDelayTime
      audio.pingPongFeedback = decodedState.pingPongFeedback
      audio.pingPongSeparation = decodedState.pingPongSeparation
      audio.pingPongGain = decodedState.pingPongGain

      // The decoder speaks Scale Workshop 2. Translate to SonicWeave.
      const sourceLines: string[] = []
      const invalidLines: [string, number][] = []
      for (let i = 0; i < decodedState.scaleLines.length; ++i) {
        const line = decodedState.scaleLines[i]
        try {
          const sourceLine = parseScaleWorkshop2Line(line, DEFAULT_NUMBER_OF_COMPONENTS).toString()
          sourceLines.push(sourceLine)
        } catch {
          invalidLines.push([line, i])
        }
      }

      annotateColors(sourceLines, decodedState.keyColors)
      for (const [line, index] of invalidLines) {
        if (!line.trim().length) {
          sourceLines.splice(index, 0, '')
        } else {
          sourceLines.splice(
            index,
            0,
            '(* ' + line.replaceAll('(*', '(\u2217').replaceAll('*)', '\u2217)') + ' *)'
          )
        }
      }
      scale.sourceText = sourceLines.join('\n')
      scale.computeScale()
      scale.history.truncate()

      // Replace query with version 3.
      await router.push({ path: getPath(url), query: { version } })
    } catch (error) {
      console.error(`Error parsing version ${query.get('version')} URL`, error)
    }
  }
  await entropy.fetchTable()
})

onUnmounted(() => {
  window.removeEventListener('keydown', windowKeydown)
  window.removeEventListener('keyup', windowKeyup)
  window.removeEventListener('keydown', windowKeydownOrUp)
  window.removeEventListener('keyup', windowKeydownOrUp)
  window.removeEventListener('mousedown', windowKeydownOrUp)
  window.removeEventListener('touchstart', audio.initialize)
  typingKeyboard.removeEventListener(typingKeydown)
  if (midi.input !== null) {
    midi.input.removeListener()
  }
  audio.unintialize()
})

function panic() {
  console.log('Firing global key off.')
  typingKeyboard.deactivate()
  midiIn.deactivate()
  if (midi.output !== null) {
    midi.output.sendAllNotesOff({
      channels: midi.outputMode === 'pitchBend' ? [...midi.outputChannels] : [midi.outputChannel]
    })
  }
  if (audio.synth !== null) {
    audio.synth.allNotesOff()
  }
}
</script>

<template>
  <nav id="app-navigation">
    <ul id="app-tabs">
      <li>
        <RouterLink to="/about"><strong>Sw</strong></RouterLink>
      </li>
      <li><RouterLink to="/">Build Scale</RouterLink></li>
      <li v-if="state.showMosTab">
        <RouterLink to="/mos">MOS</RouterLink>
      </li>
      <li><RouterLink to="/analysis">Analysis</RouterLink></li>
      <li><RouterLink to="/lattice">Lattice</RouterLink></li>
      <li><RouterLink to="/vk">Virtual Keyboard</RouterLink></li>
      <li v-if="state.showVirtualQwerty">
        <RouterLink to="/qwerty">Virtual QWERTY</RouterLink>
      </li>
      <li><RouterLink to="/synth">Synth</RouterLink></li>
      <li><RouterLink to="/midi">MIDI I/O</RouterLink></li>
      <li><RouterLink to="/prefs">Preferences</RouterLink></li>
    </ul>
    <div id="app-tray" class="hidden-sm">
      <ul>
        <template v-if="state.typingActive">
          <li title="Type on your computer keyboard to hear the synth">
            <span class="typing-info active">Key</span>
          </li>
        </template>
        <template v-else>
          <li title="Synth will not respond to keypresses until you click outside the input field">
            <span class="typing-info">Key</span>
          </li>
        </template>
      </ul>
    </div>
  </nav>
  <RouterView
    :noteOn="keyboardNoteOn"
    :midiInputChannels="midiInputChannels"
    :typingKeyboard="typingKeyboard"
    @panic="panic"
  />
  <footer id="app-footer">
    <RouterLink to="/privacy-policy">Privacy policy</RouterLink>,
    <RouterLink to="/terms-of-service">Terms of service</RouterLink>
  </footer>
</template>

<style>
@import '@/assets/base.css';
@import '@/assets/main.css';

#app {
  display: flex;
  flex-direction: column;
  align-items: stretch;
  height: 100vh;
}

nav#app-navigation {
  flex: 0 0 auto;
  display: flex;
}

#app > #view,
#app > main {
  flex: 1 1 auto;
  overflow-y: hidden;
}

/* Navigation tabs */
nav#app-navigation {
  background-color: var(--color-accent);
  color: white;
  max-width: 100%;
  overflow-x: auto;
}
ul#app-tabs {
  padding: 0px;
  margin: 0px;
  white-space: nowrap;
}
nav#app-navigation ul li {
  list-style-type: none;
  display: inline-block;
}
nav#app-navigation ul li a {
  display: inline-block;
  padding: 0.75rem 1rem;
  color: white;
  text-decoration: none;
  cursor: default;
}

nav#app-navigation ul#app-tabs li a:focus,
nav#app-navigation ul#app-tabs li a:hover {
  background-color: var(--color-accent-deeper);
}

nav#app-navigation ul#app-tabs li a.router-link-exact-active,
nav#app-navigation ul#app-tabs li a.router-link-exact-active:focus,
nav#app-navigation ul#app-tabs li a.router-link-exact-active:hover {
  background-color: var(--color-background);
  color: var(--color-text);
}

nav a.router-link-exact-active {
  color: var(--color-text);
}

nav a.router-link-exact-active:focus,
nav a.router-link-exact-active:hover {
  background-color: transparent;
}

nav a {
  display: inline-block;
  padding: 0 1rem;
  border-left: 1px solid var(--color-border);
}

nav a:first-of-type {
  border: 0;
}

/* Status indicator tray */
#app-tray {
  width: 100%;
  text-align: right;
  cursor: default;
}

#app-tray ul {
  display: inline-block;
  padding: 0.75rem 1rem;
  background-color: var(--color-accent-deeper);
}

#app-tray ul li {
  color: var(--color-accent);
}

#app-tray ul li .active {
  color: var(--color-accent-text);
}

#app-footer {
  font-size: small;
  line-height: 1;
  padding-right: 1em;
  text-align: right;
  color: var(--color-text-mute);
}
#app-footer a {
  color: var(--color-text-mute);
}
</style>