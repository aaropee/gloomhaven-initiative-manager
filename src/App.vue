<script setup>
import { computed, onBeforeUnmount, ref, watch } from 'vue'
import monsterClasses from './assets/monster-classes.json'
import scenarios from './assets/scenarios.json'

const rosterStorageKey = 'roundtracker.character-roster'
const themeStorageKey = 'roundtracker.color-theme'
const initialCharacterClasses = [
  'Scoundrel',
  'Brute',
  'Tinkerer',
  'Mindthief',
  'Cragheart',
  'Spellweaver',
]
const rosterStorageError = ref('')
const themeStorageError = ref('')
const selectedScenario = ref('')
const setupExpanded = ref(true)
const characterClassDraft = ref('')
const characterClassError = ref('')
const themes = [
  { value: 'default', label: 'Default' },
  { value: 'elegant', label: 'Elegant' },
  { value: 'spring', label: 'Spring' },
  { value: 'harvest', label: 'Harvest' },
  { value: 'fall', label: 'Fall' },
  { value: 'frost', label: 'Frost' },
]
const storageError = computed(() =>
  [rosterStorageError.value, themeStorageError.value].filter(Boolean).join(' '),
)

function loadSelectedTheme() {
  try {
    const savedTheme = localStorage.getItem(themeStorageKey)
    return themes.some((theme) => theme.value === savedTheme) ? savedTheme : 'default'
  } catch (error) {
    themeStorageError.value = 'Could not load the saved theme. The default theme will be used.'
    return 'default'
  }
}

const selectedTheme = ref(loadSelectedTheme())
const normalizeMonsterName = (name) => name.trim().replace(/\s+/g, ' ')
const monsterClassByName = new Map(
  Object.entries(monsterClasses).map(([name, monsterClass]) => [
    normalizeMonsterName(name),
    monsterClass,
  ]),
)

function createDefaultCharacters() {
  return []
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
        active: true,
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
const manualGroupOrders = ref({ active: null, inactive: null })
const draggedParticipantId = ref(null)
let nextId = Math.max(0, ...participants.value.map((participant) => participant.id)) + 1

function participantGroup(participant) {
  return participant.active ? 'active' : 'inactive'
}

function applyManualOrder(groupParticipants, manualOrder) {
  if (!manualOrder) return groupParticipants

  const orderIndex = new Map(manualOrder.map((id, index) => [id, index]))
  return [...groupParticipants].sort((a, b) => {
    const aIndex = orderIndex.get(a.id)
    const bIndex = orderIndex.get(b.id)
    if (aIndex === undefined && bIndex === undefined) return 0
    if (aIndex === undefined) return 1
    if (bIndex === undefined) return -1
    return aIndex - bIndex
  })
}

const initiativeOrderedParticipants = computed(() =>
  [...participants.value].sort((a, b) => {
    const aHasInitiative = Number.isFinite(a.initiative)
    const bHasInitiative = Number.isFinite(b.initiative)

    if (aHasInitiative && bHasInitiative) {
      if (a.initiative !== b.initiative) return a.initiative - b.initiative
      if (a.type !== b.type) return a.type === 'character' ? -1 : 1
    }

    if (aHasInitiative) return -1
    if (bHasInitiative) return 1
    return participants.value.indexOf(a) - participants.value.indexOf(b)
  }),
)

const activeParticipants = computed(() =>
  applyManualOrder(
    initiativeOrderedParticipants.value.filter(
      (participant) => participantGroup(participant) === 'active',
    ),
    manualGroupOrders.value.active,
  ),
)
const inactiveParticipants = computed(() =>
  applyManualOrder(
    participants.value.filter(
      (participant) => participantGroup(participant) === 'inactive',
    ),
    manualGroupOrders.value.inactive,
  ),
)
const participantGroups = computed(() => [
  { key: 'active', label: 'Active', participants: activeParticipants.value },
  { key: 'inactive', label: 'Inactive', participants: inactiveParticipants.value },
])

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

watch(selectedTheme, (theme) => {
  try {
    localStorage.setItem(themeStorageKey, theme)
    themeStorageError.value = ''
  } catch (error) {
    themeStorageError.value = 'Could not save the selected theme on this device.'
  }
})

function addParticipant(type) {
  participants.value.push({
    id: nextId++,
    type,
    source: 'manual',
    name: 'New monster',
    detail: '',
    active: true,
    initiative: null,
    initiativeDraft: '',
    acted: false,
  })
}

function addCharacter() {
  characterClassError.value = ''
  const enteredClass = characterClassDraft.value.trim().replace(/\s+/g, ' ')
  if (!enteredClass) return

  const characterClass =
    initialCharacterClasses.find((name) => name.toLowerCase() === enteredClass.toLowerCase()) ||
    enteredClass
  const alreadyAdded = participants.value.some(
    (participant) =>
      participant.type === 'character' &&
      participant.name.trim().toLowerCase() === characterClass.toLowerCase(),
  )

  if (alreadyAdded) {
    characterClassError.value = `${characterClass} is already in the character roster.`
    return
  }

  participants.value.push({
    id: nextId++,
    type: 'character',
    source: 'manual',
    name: characterClass,
    detail: '',
    active: true,
    initiative: null,
    initiativeDraft: '',
    acted: false,
  })

  characterClassDraft.value = ''
}

function loadScenarioMonsters(scenarioName) {
  const scenario = scenarios.find(({ name }) => name === scenarioName)
  if (!scenario) throw new Error(`Unknown scenario: ${scenarioName}`)

  const existingParticipants = participants.value.filter(
    (participant) => participant.source !== 'scenario',
  )
  const scenarioMonsters = scenario.monsters.map((monster) => {
    const name = normalizeMonsterName(monster.name)
    const detail = monsterClassByName.get(name)
    if (!detail) throw new Error(`Missing class mapping for monster: ${monster.name}`)

    return {
      type: 'monster',
      source: 'scenario',
      name,
      detail,
      active: false,
      initiative: null,
      initiativeDraft: '',
      acted: false,
    }
  })

  const identifiedScenarioMonsters = scenarioMonsters.map((monster) => ({
    id: nextId++,
    ...monster,
  }))

  selectedScenario.value = scenarioName
  participants.value = [...existingParticipants, ...identifiedScenarioMonsters]
}

function toggleActed(participant) {
  participant.acted = !participant.acted
}

function commitInitiative(participant) {
  const value = participant.initiativeDraft.trim()
  const previousInitiative = participant.initiative

  if (!value) {
    participant.initiative = null
    return
  }

  const initiative = Number(value)
  if (Number.isFinite(initiative) && initiative !== previousInitiative) {
    participant.initiative = initiative
  }
}

function startNextRound() {
  round.value += 1
  manualGroupOrders.value = { ...manualGroupOrders.value, active: null }
  participants.value.forEach((participant) => {
    participant.initiative = null
    participant.initiativeDraft = ''
    participant.acted = false
  })
}

function removeParticipant(id) {
  const participant = participants.value.find((entry) => entry.id === id)
  if (!participant || participant.type === 'character') return

  participants.value = participants.value.filter((entry) => entry.id !== id)
  manualGroupOrders.value = {
    active: manualGroupOrders.value.active?.filter((entryId) => entryId !== id) ?? null,
    inactive: manualGroupOrders.value.inactive?.filter((entryId) => entryId !== id) ?? null,
  }
}

function moveParticipantToGroup(participant, group, targetParticipant = null) {
  if (!participant) return

  const previousGroup = participantGroup(participant)
  const active = group === 'active'
  if (previousGroup === group) return

  participant.active = active
  participant.initiative = null
  participant.initiativeDraft = ''
  participant.acted = false

  const nextOrders = {
    ...manualGroupOrders.value,
    [previousGroup]: manualGroupOrders.value[previousGroup]?.filter(
      (id) => id !== participant.id,
    ) ?? null,
  }

  if (targetParticipant && participantGroup(targetParticipant) === group) {
    const newGroupOrder = (group === 'active' ? activeParticipants.value : inactiveParticipants.value)
      .filter((entry) => entry.id !== participant.id)
      .map((entry) => entry.id)
    const targetIndex = newGroupOrder.indexOf(targetParticipant.id)
    newGroupOrder.splice(targetIndex === -1 ? newGroupOrder.length : targetIndex, 0, participant.id)
    nextOrders[group] = newGroupOrder
  } else if (nextOrders[group]) {
    nextOrders[group] = [
      ...nextOrders[group].filter((id) => id !== participant.id),
      participant.id,
    ]
  }

  manualGroupOrders.value = nextOrders
}

function toggleParticipantGroup(participant) {
  moveParticipantToGroup(participant, participant.active ? 'inactive' : 'active')
}

function reorderParticipants(draggedParticipant, targetParticipant, group) {
  if (
    !draggedParticipant ||
    !targetParticipant ||
    targetParticipant.id === draggedParticipant.id ||
    participantGroup(draggedParticipant) !== group ||
    participantGroup(targetParticipant) !== group
  ) {
    return
  }

  const groupOrder = (group === 'active' ? activeParticipants.value : inactiveParticipants.value)
    .map((participant) => participant.id)
  const fromIndex = groupOrder.indexOf(draggedParticipant.id)
  const toIndex = groupOrder.indexOf(targetParticipant.id)
  if (fromIndex === -1 || toIndex === -1) return

  groupOrder.splice(fromIndex, 1)
  groupOrder.splice(toIndex, 0, draggedParticipant.id)
  manualGroupOrders.value = { ...manualGroupOrders.value, [group]: groupOrder }
}

function moveParticipantWithinGroup(participant, direction) {
  const group = participantGroup(participant)
  const groupParticipants = group === 'active' ? activeParticipants.value : inactiveParticipants.value
  const currentIndex = groupParticipants.findIndex((entry) => entry.id === participant.id)
  const targetParticipant = groupParticipants[currentIndex + direction]
  if (targetParticipant) reorderParticipants(participant, targetParticipant, group)
}

function canDragParticipant(participant) {
  return participant.type === 'character' || participant.type === 'monster'
}

function isGroupDropTarget(group) {
  const draggedParticipant = participants.value.find(
    (participant) => participant.id === draggedParticipantId.value,
  )

  return (
    draggedParticipant?.type === 'monster' &&
    (group === 'active' ? !draggedParticipant.active : draggedParticipant.active)
  )
}

function startParticipantDrag(event, participant) {
  if (event.button !== 0 || !canDragParticipant(participant)) return

  event.preventDefault()
  draggedParticipantId.value = participant.id
  event.currentTarget.setPointerCapture(event.pointerId)
}

function finishParticipantDrag(event) {
  if (draggedParticipantId.value === null) return

  const draggedParticipant = participants.value.find(
    (participant) => participant.id === draggedParticipantId.value,
  )
  const target = document.elementFromPoint(event.clientX, event.clientY)
  const targetCard = target?.closest('.initiative-card')
  const targetId = Number(targetCard?.dataset.participantId)
  const targetParticipant = participants.value.find(
    (participant) => participant.id === targetId,
  )
  const targetGroup = target?.closest('[data-participant-group]')?.dataset.participantGroup

  if (
    draggedParticipant?.type === 'monster' &&
    (targetGroup === 'active' || targetGroup === 'inactive') &&
    targetGroup !== (draggedParticipant.active ? 'active' : 'inactive')
  ) {
    moveParticipantToGroup(draggedParticipant, targetGroup, targetParticipant)
  } else {
    reorderParticipants(draggedParticipant, targetParticipant, targetGroup)
  }
  draggedParticipantId.value = null
}

function cancelTieDrag() {
  draggedParticipantId.value = null
}

onBeforeUnmount(cancelTieDrag)
</script>

<template>
  <div class="app-shell" :data-theme="selectedTheme">
    <header class="topbar">
      <a class="brand" href="#" aria-label="Roundtracker home">
        <span class="brand-mark" style="width: 100%; padding: 0 7px;" aria-hidden="true">Roundtracker</span>
        <span style="text-transform: uppercase; display: none;">Roundtracker</span>
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
          <h1 id="page-title">Order of Initiative</h1>
          <p class="intro">
            Enter initiatives as revealed, then drag any card to set the order until the next round.
          </p>
        </div>
      </section>

      <section class="setup-accordion" aria-label="Scenario and character setup">
        <button
          id="setup-accordion-toggle"
          class="setup-accordion-toggle"
          type="button"
          aria-controls="setup-accordion-content"
          :aria-expanded="setupExpanded"
          @click="setupExpanded = !setupExpanded"
        >
          <span>Scenario and characters</span>
          <span class="setup-toggle-label">{{ setupExpanded ? 'Hide setup' : 'Edit setup' }}</span>
          <span class="setup-toggle-icon" aria-hidden="true">{{ setupExpanded ? '−' : '+' }}</span>
        </button>
        <div
          v-show="setupExpanded"
          id="setup-accordion-content"
          class="setup-accordion-content"
          aria-labelledby="setup-accordion-toggle"
        >
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
          </div>
          <form class="character-picker" @submit.prevent="addCharacter">
            <label for="character-class">Player character class</label>
            <div class="character-picker-input">
              <input
                id="character-class"
                v-model="characterClassDraft"
                type="text"
                list="character-class-options"
                maxlength="36"
                placeholder="Choose or enter a class"
                autocomplete="off"
                required
                @input="characterClassError = ''"
              />
              <datalist id="character-class-options">
                <option v-for="name in initialCharacterClasses" :key="name" :value="name" />
              </datalist>
              <button class="add-button" type="submit">
                <span aria-hidden="true">＋</span> Add character
              </button>
            </div>
            <p v-if="characterClassError" class="character-class-error" role="status">
              {{ characterClassError }}
            </p>
          </form>
        </div>
      </section>

      <section class="scenario-summary-row" aria-label="Current scenario and round">
        <div v-if="selectedScenario" class="scenario-summary">
          <span>SCENARIO</span>
          <strong>{{ selectedScenario }}</strong>
        </div>
        <div class="scenario-summary-actions">
          <div class="round-token" aria-label="Current round">
            <span class="round-token-label">ROUND</span>
            <span class="round-number">{{ String(round).padStart(2, '0') }}</span>
          </div>
          <button class="next-round-button" type="button" @click="startNextRound">
            <span>New round</span>
            <span aria-hidden="true">↗</span>
          </button>
        </div>
      </section>

      <section
        v-for="group in participantGroups"
        :key="group.key"
        class="participant-group"
        :class="[`is-${group.key}-group`, { 'is-drop-target': isGroupDropTarget(group.key) }]"
      >
        <header class="participant-group-heading">
          <h2>{{ group.label }}</h2>
          <span>{{ group.participants.length }}</span>
        </header>
        <div
          class="initiative-list"
          :data-participant-group="group.key"
          :aria-label="`${group.label} participants`"
        >
          <article
            v-for="(participant, index) in group.participants"
            :key="participant.id"
            class="initiative-card"
            :class="[
              `is-${participant.type}`,
              {
                'is-inactive': group.key === 'inactive',
                'has-initiative': Number.isFinite(participant.initiative),
                'is-acted': participant.acted,
                'is-dragging': draggedParticipantId === participant.id,
              },
            ]"
            :data-participant-id="participant.id"
          >
            <button
              class="order-number drag-handle"
              type="button"
              :disabled="!canDragParticipant(participant)"
              :aria-label="
                participant.type === 'monster'
                  ? `Drag ${participant.name} to reorder or move between groups`
                  : `Drag ${participant.name} to reorder active cards`
              "
              aria-keyshortcuts="ArrowUp ArrowDown"
              @pointerdown="startParticipantDrag($event, participant)"
              @pointerup="finishParticipantDrag"
              @pointercancel="cancelTieDrag"
              @keydown.up.prevent="moveParticipantWithinGroup(participant, -1)"
              @keydown.down.prevent="moveParticipantWithinGroup(participant, 1)"
            >
              <span>{{ String(index + 1).padStart(2, '0') }}</span>
              <span v-if="canDragParticipant(participant)" class="drag-grip" aria-hidden="true">
                ⠿
              </span>
            </button>
            <div class="participant-info">
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
            <label
              v-if="participant.active"
              class="initiative-entry"
            >
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
              v-if="participant.active"
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
              class="group-toggle-button"
              type="button"
              :aria-label="`Move ${participant.name} to ${participant.active ? 'inactive' : 'active'}`"
              @click="toggleParticipantGroup(participant)"
            >
              <span aria-hidden="true">{{ participant.active ? '↓' : '↑' }}</span>
            </button>
          </article>

          <div v-if="!group.participants.length" class="group-empty-state">
            {{
              group.key === 'active'
                ? 'Drag revealed monsters here to add them to active combat.'
                : 'Scenario monsters will appear here until they are revealed.'
            }}
          </div>
        </div>
      </section>

      <section class="add-controls" aria-label="Add participants">
        <button class="add-button" type="button" @click="addParticipant('monster')">
          <span aria-hidden="true">＋</span> Add monster group
        </button>
      </section>

      <p v-if="storageError" class="footer-note storage-warning" role="status">
        {{ storageError }}
      </p>
      <p v-else class="footer-note">
        Character names, classes, and theme save on this device. Monster groups and round status reset on refresh.
      </p>
    </main>
  </div>
</template>
