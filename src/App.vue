<script setup>
import { computed, ref, watch } from 'vue'
import scenarios from './assets/scenarios.json'

const rosterStorageKey = 'roundkeeper.character-roster'
const rosterStorageError = ref('')
const selectedTheme = ref('default')
const selectedScenario = ref('')
const themes = [
  { value: 'default', label: 'Default' },
  { value: 'elegant', label: 'Elegant' },
  { value: 'spring', label: 'Spring' },
  { value: 'harvest', label: 'Harvest' },
  { value: 'fall', label: 'Fall' },
  { value: 'frost', label: 'Frost' },
]

function createDefaultCharacters() {
  return Array.from({ length: 4 }, (_, index) => ({
    id: index + 1,
    type: 'character',
    name: `Character ${index + 1}`,
    detail: '',
    initiative: null,
    initiativeDraft: '',
    acted: false,
  }))
}

function loadCharacters() {
  try {
    const savedRoster = localStorage.getItem(rosterStorageKey)
    if (!savedRoster) return createDefaultCharacters()

    const parsedRoster = JSON.parse(savedRoster)
    if (!Array.isArray(parsedRoster)) throw new Error('Saved roster is not an array.')

    return parsedRoster.map((character, index) => {
      if (
        !character ||
        typeof character.name !== 'string' ||
        typeof character.detail !== 'string'
      ) {
        throw new Error('Saved roster contains an invalid character.')
      }

      return {
        id: index + 1,
        type: 'character',
        name: character.name,
        detail: character.detail,
        initiative: null,
        initiativeDraft: '',
        acted: false,
      }
    })
  } catch (error) {
    rosterStorageError.value = 'Could not load the saved character roster. Changes may not persist on this device.'
    return createDefaultCharacters()
  }
}

const round = ref(1)
const participants = ref(loadCharacters())
let nextId = Math.max(0, ...participants.value.map((participant) => participant.id)) + 1

const orderedParticipants = computed(() =>
  [...participants.value].sort((a, b) => {
    const aHasInitiative = Number.isFinite(a.initiative)
    const bHasInitiative = Number.isFinite(b.initiative)

    if (aHasInitiative && bHasInitiative) return a.initiative - b.initiative
    if (aHasInitiative) return -1
    if (bHasInitiative) return 1
    return participants.value.indexOf(a) - participants.value.indexOf(b)
  }),
)

const assignedCount = computed(
  () => participants.value.filter((participant) => Number.isFinite(participant.initiative)).length,
)

const savedCharacters = computed(() =>
  participants.value
    .filter((participant) => participant.type === 'character')
    .map(({ name, detail }) => ({ name, detail })),
)

watch(
  savedCharacters,
  (characters) => {
    try {
      localStorage.setItem(rosterStorageKey, JSON.stringify(characters))
      rosterStorageError.value = ''
    } catch (error) {
      rosterStorageError.value = 'Could not save the character roster. Check this browser’s storage settings.'
    }
  },
  { deep: true },
)

function addParticipant(type) {
  const isCharacter = type === 'character'
  participants.value.push({
    id: nextId++,
    type,
    source: 'manual',
    name: isCharacter ? 'New character' : 'New monster',
    detail: '',
    initiative: null,
    initiativeDraft: '',
    acted: false,
  })
}

function loadScenarioMonsters(scenarioName) {
  const scenario = scenarios.find(({ name }) => name === scenarioName)
  if (!scenario) throw new Error(`Unknown scenario: ${scenarioName}`)

  selectedScenario.value = scenarioName
  const existingParticipants = participants.value.filter(
    (participant) => participant.source !== 'scenario',
  )
  const scenarioMonsters = scenario.monsters.map((monster) => ({
    id: nextId++,
    type: 'monster',
    source: 'scenario',
    name: monster.name,
    detail: '',
    initiative: null,
    initiativeDraft: '',
    acted: false,
  }))

  participants.value = [...existingParticipants, ...scenarioMonsters]
}

function toggleActed(participant) {
  participant.acted = !participant.acted
}

function commitInitiative(participant) {
  const value = participant.initiativeDraft.trim()

  if (!value) {
    participant.initiative = null
    return
  }

  const initiative = Number(value)
  if (Number.isFinite(initiative)) participant.initiative = initiative
}

function startNextRound() {
  round.value += 1
  participants.value.forEach((participant) => {
    participant.initiative = null
    participant.initiativeDraft = ''
    participant.acted = false
  })
}

function clearInitiatives() {
  participants.value.forEach((participant) => {
    participant.initiative = null
    participant.initiativeDraft = ''
  })
}

function removeParticipant(id) {
  participants.value = participants.value.filter((participant) => participant.id !== id)
}
</script>

<template>
  <div class="app-shell" :data-theme="selectedTheme">
    <header class="topbar">
      <a class="brand" href="#" aria-label="Roundkeeper home">
        <span class="brand-mark" aria-hidden="true">R</span>
        <span>ROUNDKEEPER</span>
      </a>
      <label class="theme-picker" for="theme-select">
        <span class="theme-picker-label">Theme</span>
        <select id="theme-select" v-model="selectedTheme" aria-label="Color theme">
          <option v-for="theme in themes" :key="theme.value" :value="theme.value">
            {{ theme.label }}
          </option>
        </select>
      </label>
    </header>

    <main class="tracker">
      <section class="round-heading" aria-labelledby="page-title">
        <div>
          <!-- <p class="eyebrow">CAMPAIGN TABLE</p> -->
          <h1 id="page-title">Initiative order</h1>
          <p class="intro">Enter each initiative as it’s revealed. The order updates automatically.</p>
        </div>
        <div class="round-token" aria-label="Current round">
          <span class="round-token-label">ROUND</span>
          <span class="round-number">{{ String(round).padStart(2, '0') }}</span>
        </div>
      </section>

      <section class="round-controls" aria-label="Scenario and round controls">
        <div class="scenario-picker">
          <label for="scenario-select">Scenario</label>
          <select
            id="scenario-select"
            v-model="selectedScenario"
            aria-label="Choose a scenario"
            @change="loadScenarioMonsters($event.target.value)"
          >
            <option value="" disabled>Select a scenario</option>
            <option v-for="scenario in scenarios" :key="scenario.name" :value="scenario.name">
              {{ scenario.name }}
            </option>
          </select>
          <p class="scenario-help">
            Changing scenarios replaces their generated monsters, while keeping characters and manually added monsters.
          </p>
        </div>
        <div class="progress-label">
          <span>{{ assignedCount }} <span class="muted">/ {{ participants.length }} initiatives</span></span>
          <span v-if="assignedCount === participants.length && participants.length" class="ready-label">
            Order ready
          </span>
          <span v-else class="muted">Lowest initiative goes first</span>
        </div>
        <div class="progress-track" aria-hidden="true">
          <span :style="{ width: `${participants.length ? (assignedCount / participants.length) * 100 : 0}%` }"></span>
        </div>
        <div class="control-actions">
          <button class="text-button" type="button" @click="clearInitiatives">Clear values</button>
          <button class="next-round-button" type="button" @click="startNextRound">
            <span>New round</span>
            <span aria-hidden="true">↗</span>
          </button>
        </div>
      </section>

      <section class="initiative-list" aria-label="Initiative cards">
        <article
          v-for="(participant, index) in orderedParticipants"
          :key="participant.id"
          class="initiative-card"
          :class="[
            `is-${participant.type}`,
            {
              'has-initiative': Number.isFinite(participant.initiative),
              'is-acted': participant.acted,
            },
          ]"
        >
          <div class="order-number" aria-hidden="true">{{ String(index + 1).padStart(2, '0') }}</div>
          <div class="participant-info">
            <span class="type-label">{{ participant.type === 'character' ? 'PLAYER CHARACTER' : 'MONSTER GROUP' }}</span>
            <input
              v-model="participant.name"
              class="name-input"
              :aria-label="`${participant.type === 'character' ? 'Character' : 'Monster group'} name`"
              maxlength="36"
              @focus="$event.target.select()"
            />
            <input
              v-if="participant.type === 'character'"
              v-model="participant.detail"
              class="detail-input"
              aria-label="Character class or notes"
              placeholder="Class / notes"
              maxlength="36"
            />
            <input
              v-else
              v-model="participant.detail"
              class="detail-input"
              aria-label="Monster group notes"
              placeholder="Optional notes"
              maxlength="36"
            />
          </div>
          <label class="initiative-entry">
            <span>INITIATIVE</span>
            <input
              :value="participant.initiativeDraft"
              type="number"
              inputmode="numeric"
              min="1"
              max="99"
              step="1"
              placeholder="—"
              :aria-label="`Initiative for ${participant.name || 'unnamed participant'}`"
              @input="participant.initiativeDraft = $event.target.value"
              @focus="$event.target.select()"
              @blur="commitInitiative(participant)"
              @keydown.enter.prevent="$event.target.blur()"
            />
          </label>
          <button
            class="acted-button"
            :class="{ 'is-marked': participant.acted }"
            type="button"
            :aria-pressed="participant.acted"
            :aria-label="participant.acted ? `Mark ${participant.name} as not acted` : `Mark ${participant.name} as acted`"
            @click="toggleActed(participant)"
          >
            <span class="acted-check" aria-hidden="true">{{ participant.acted ? '✓' : '' }}</span>
            <span>{{ participant.acted ? 'Done' : 'End turn' }}</span>
          </button>
          <button
            class="remove-button"
            type="button"
            :aria-label="`Remove ${participant.name || 'participant'}`"
            @click="removeParticipant(participant.id)"
          >
            <span aria-hidden="true">×</span>
          </button>
        </article>

        <div v-if="!participants.length" class="empty-state">
          <span class="empty-icon" aria-hidden="true">＋</span>
          <h2>Your initiative row is empty</h2>
          <p>Add player characters and monster groups to get started.</p>
        </div>
      </section>

      <section class="add-controls" aria-label="Add participants">
        <button class="add-button" type="button" @click="addParticipant('character')">
          <span aria-hidden="true">＋</span> Add character
        </button>
        <button class="add-button" type="button" @click="addParticipant('monster')">
          <span aria-hidden="true">＋</span> Add monster group
        </button>
      </section>

      <p v-if="rosterStorageError" class="footer-note storage-warning" role="status">
        {{ rosterStorageError }}
      </p>
      <p v-else class="footer-note">
        Character names and classes save on this device. Monster groups and round status reset on refresh.
      </p>
    </main>
  </div>
</template>
