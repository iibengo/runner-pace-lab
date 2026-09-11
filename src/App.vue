<script setup lang="ts">
import { computed, ref, watch } from 'vue'

type Strategy = 'constant' | 'negative' | 'elevation'
type MilestoneType = 'gel' | 'water' | 'uphill' | 'downhill' | 'crowd' | 'pace' | 'finish' | 'note'
type Effort = 'Cómodo' | 'Controlado' | 'Moderado' | 'Exigente' | 'Sprint'

interface Milestone {
  id: number
  km: number
  type: MilestoneType
  note: string
}

interface PacePoint {
  km: number
  pace: number
  elapsed: number
  effort: Effort
}

interface SavedPlan {
  id: number
  name: string
  distance: number
  targetTime: number
  strategy: Strategy
  date: string
  milestones: Milestone[]
  paceData: PacePoint[]
}

const distance = ref(42.195)
const targetTime = ref(210)
const strategy = ref<Strategy>('negative')
const tableMode = ref<'1' | '5' | 'checkpoints'>('5')
const milestones = ref<Milestone[]>([
  { id: 1, km: 5, type: 'water', note: '' },
  { id: 2, km: 12, type: 'uphill', note: '' },
  { id: 3, km: 20, type: 'gel', note: 'Gel + agua' },
  { id: 4, km: 30, type: 'crowd', note: '' },
  { id: 5, km: 38, type: 'pace', note: '' },
  { id: 6, km: 42.195, type: 'finish', note: 'Meta' },
])
const savedPlans = ref<SavedPlan[]>(loadSavedPlans())
const selectedKilometer = ref<number | null>(null)
const showMilestoneMenu = ref(false)
const selectedMilestoneType = ref<MilestoneType>('gel')
const milestoneNote = ref('')
const showSavedPlans = ref(false)
const showComparison = ref(false)
const compareA = ref<number | null>(null)
const compareB = ref<number | null>(null)
const nextId = ref(10)

const strategyLabel = computed(() => ({ constant: 'Ritmo Constante', negative: 'Negative Split', elevation: 'Ajustado por Desnivel' }[strategy.value]))
const elevationProfile = computed(() => generateElevationProfile())
const paceData = computed(() => generateRacePlan())
const averagePace = computed(() => targetTime.value / distance.value)
const formattedTargetTime = computed(() => formatTime(targetTime.value * 60))
const nutritionPlan = computed(() => milestones.value.filter((milestone) => milestone.type === 'gel'))
const checkpoints = computed(() => generateCheckpoints())
const difficulty = computed(() => strategyDifficulty.value)
const strategyDifficulty = computed(() => {
  const base = targetTime.value / distance.value
  if (base < 4.5) return 'Alta'
  if (base < 5.75) return strategy.value === 'constant' ? 'Media' : 'Media-alta'
  return 'Controlada'
})
const strategyScore = computed(() => {
  const spread = Math.max(...paceData.value.map((point) => point.pace)) - Math.min(...paceData.value.map((point) => point.pace))
  const nutrition = nutritionPlan.value.length > 0 ? 12 : 0
  const distribution = nutritionPlan.value.filter((gel) => gel.km > distance.value * 0.12 && gel.km < distance.value * 0.92).length * 3
  return Math.max(48, Math.min(99, Math.round(96 - spread * 9 + nutrition + distribution - (strategy.value === 'negative' ? 2 : 0))))
})
const alerts = computed(() => getSmartAlerts())
const remainingDistance = computed(() => Math.max(0, distance.value - (milestones.value.length ? Math.max(...milestones.value.map((item) => item.km)) : 0)))
const finalPace = computed(() => paceData.value[paceData.value.length - 1]?.pace ?? averagePace.value)
const tableRows = computed(() => {
  if (tableMode.value === '1') return paceData.value
  if (tableMode.value === 'checkpoints') return checkpoints.value.map((checkpoint) => paceData.value.find((point) => Math.abs(point.km - checkpoint) < 0.01) ?? closestPoint(checkpoint))
  return paceData.value.filter((point) => point.km % 5 < 0.01 || point.km === distance.value)
})
const comparePlans = computed(() => {
  const first = savedPlans.value.find((plan) => plan.id === compareA.value)
  const second = savedPlans.value.find((plan) => plan.id === compareB.value)
  return first && second ? { first, second } : null
})
const chartWidth = 900
const chartHeight = 290
const elevationHeight = 170

watch([distance, targetTime], () => {
  milestones.value = milestones.value.map((milestone) => ({ ...milestone, km: Math.min(milestone.km, distance.value) }))
}, { immediate: true })

function calculateAveragePace(): number {
  return targetTime.value / distance.value
}

function generateConstantPace(): number[] {
  return Array.from({ length: Math.ceil(distance.value) }, () => calculateAveragePace())
}

function generateNegativeSplit(): number[] {
  const weights = [1.045, 1.025, 1, 0.975, 0.94]
  const raw = Array.from({ length: Math.ceil(distance.value) }, (_, index) => {
    const ratio = (index + 0.5) / distance.value
    const phase = ratio < 0.24 ? 0 : ratio < 0.5 ? 1 : ratio < 0.72 ? 2 : ratio < 0.9 ? 3 : 4
    return weights[phase]
  })
  const weightedAverage = raw.reduce((sum, value) => sum + value, 0) / raw.length
  return raw.map((value) => calculateAveragePace() * value / weightedAverage)
}

function generateElevationAdjustedPace(): number[] {
  const constant = generateConstantPace()
  const adjusted = constant.map((pace, index) => {
    const km = Math.min(index + 0.5, distance.value)
    const slope = getSlopeAt(km)
    return pace * (1 + slope * 0.00038)
  })
  const total = adjusted.reduce((sum, value, index) => sum + value * (index === adjusted.length - 1 ? distance.value - index : 1), 0)
  const factor = (calculateAveragePace() * distance.value) / total
  return adjusted.map((pace) => pace * factor)
}

function generateElevationProfile(): number[] {
  const points = 70
  return Array.from({ length: points }, (_, index) => {
    const ratio = index / (points - 1)
    const base = 120 + Math.sin(ratio * Math.PI * 2.7) * 18 + Math.sin(ratio * Math.PI * 9) * 7
    const hill = Math.exp(-Math.pow((ratio - 0.58) / 0.12, 2)) * 115
    const descent = Math.exp(-Math.pow((ratio - 0.76) / 0.1, 2)) * 58
    return Math.round(base + hill - descent + (ratio > 0.86 ? 8 : 0))
  })
}

function generateCheckpoints(): number[] {
  return [5, 10, 21, 30, distance.value].filter((km, index, values) => km <= distance.value && values.indexOf(km) === index)
}

function generateRacePlan(): PacePoint[] {
  const paces = strategy.value === 'constant' ? generateConstantPace() : strategy.value === 'negative' ? generateNegativeSplit() : generateElevationAdjustedPace()
  const points: PacePoint[] = []
  let elapsed = 0
  paces.forEach((pace, index) => {
    const km = Math.min(index + 1, distance.value)
    const segment = index === paces.length - 1 ? distance.value - index : 1
    elapsed += pace * segment
    points.push({ km, pace, elapsed, effort: getEffortLevel(pace, index, paces.length) })
  })
  const correction = targetTime.value / (elapsed || 1)
  return points.map((point) => ({ ...point, pace: point.pace * correction, elapsed: point.elapsed * correction }))
}

function formatTime(seconds: number): string {
  const rounded = Math.max(0, Math.round(seconds))
  const hours = Math.floor(rounded / 3600)
  const minutes = Math.floor((rounded % 3600) / 60)
  const remaining = rounded % 60
  return hours > 0 ? `${hours}h ${String(minutes).padStart(2, '0')}m ${String(remaining).padStart(2, '0')}s` : `${minutes}m ${String(remaining).padStart(2, '0')}s`
}

function formatCompactTime(seconds: number): string {
  const rounded = Math.max(0, Math.round(seconds))
  const hours = Math.floor(rounded / 3600)
  const minutes = Math.floor((rounded % 3600) / 60)
  const remaining = rounded % 60
  return hours > 0 ? `${hours}:${String(minutes).padStart(2, '0')}:${String(remaining).padStart(2, '0')}` : `${minutes}:${String(remaining).padStart(2, '0')}`
}

function formatPace(minutes: number): string {
  const totalSeconds = Math.round(minutes * 60)
  return `${Math.floor(totalSeconds / 60)}:${String(totalSeconds % 60).padStart(2, '0')}`
}

function getEffortLevel(pace: number, index = 0, total = 1): Effort {
  const ratio = pace / averagePace.value
  if (strategy.value === 'negative' && index > total * 0.9) return 'Sprint'
  if (ratio < 0.91) return 'Sprint'
  if (ratio < 0.97) return 'Exigente'
  if (ratio < 1.02) return 'Moderado'
  if (ratio < 1.07) return 'Controlado'
  return 'Cómodo'
}

function effortPillClass(effort: Effort): string {
  const map: Record<Effort, string> = {
    'Cómodo': 'bg-teal-50 text-teal-700',
    'Controlado': 'bg-teal-50 text-teal-700',
    'Moderado': 'bg-amber-100 text-amber-700',
    'Exigente': 'bg-orange-100 text-orange-700',
    'Sprint': 'bg-red-100 text-red-700',
  }
  return map[effort]
}

function getSlopeAt(km: number): number {
  const index = Math.min(elevationProfile.value.length - 2, Math.max(0, Math.round((km / distance.value) * (elevationProfile.value.length - 1))))
  return elevationProfile.value[index + 1] - elevationProfile.value[index]
}

function closestPoint(km: number): PacePoint {
  return paceData.value.reduce((closest, point) => Math.abs(point.km - km) < Math.abs(closest.km - km) ? point : closest, paceData.value[0])
}

function milestoneIcon(type: MilestoneType): string {
  return { gel: 'GEL', water: 'H2O', uphill: 'UP', downhill: 'DOWN', crowd: 'FAN', pace: 'PACE', finish: 'META', note: 'NOTA' }[type]
}

function milestoneLabel(type: MilestoneType): string {
  return { gel: 'Gel / Nutrición', water: 'Hidratación', uphill: 'Inicio de subida', downhill: 'Fin de subida', crowd: 'Animación / Familia', pace: 'Cambio de ritmo', finish: 'Objetivo', note: 'Nota' }[type]
}

function getSmartAlerts(): string[] {
  const result: string[] = []
  milestones.value.forEach((milestone) => {
    if (milestone.type === 'gel' && getSlopeAt(milestone.km) > 25) result.push(`El gel del km ${formatKm(milestone.km)} coincide con una subida exigente. Tómalo 1–2 km antes.`)
    if (milestone.type === 'pace' && milestone.km < distance.value * 0.65) result.push(`El cambio de ritmo del km ${formatKm(milestone.km)} parece demasiado temprano para esta estrategia.`)
    if (milestone.type === 'pace' && milestone.km < distance.value * 0.9) result.push('Reserva el sprint final para los últimos kilómetros para hacerlo más sostenible.')
  })
  return [...new Set(result)]
}

function formatKm(km: number): string {
  return Number.isInteger(km) ? String(km) : km.toFixed(1)
}

function addMilestone(type = selectedMilestoneType.value): void {
  if (selectedKilometer.value === null) return
  milestones.value.push({ id: nextId.value++, km: Math.min(distance.value, Math.max(0.1, selectedKilometer.value)), type, note: milestoneNote.value.trim() })
  milestoneNote.value = ''
  showMilestoneMenu.value = false
}

function removeMilestone(id: number): void {
  milestones.value = milestones.value.filter((milestone) => milestone.id !== id)
}

function handleChartClick(event: MouseEvent): void {
  const svg = event.currentTarget as SVGElement
  const box = svg.getBoundingClientRect()
  const x = Math.max(0, Math.min(box.width, event.clientX - box.left))
  selectedKilometer.value = Math.round((x / box.width) * distance.value * 10) / 10
  showMilestoneMenu.value = true
}

function generateNutritionPlan(): void {
  milestones.value = milestones.value.filter((milestone) => milestone.type !== 'gel')
  const interval = targetTime.value <= 120 ? 35 : 42
  for (let minutes = interval; minutes < targetTime.value - 20; minutes += interval) {
    const point = closestPoint((minutes / targetTime.value) * distance.value)
    const adjustedKm = getSlopeAt(point.km) > 25 ? Math.max(1, point.km - 1.2) : point.km
    milestones.value.push({ id: nextId.value++, km: Math.round(adjustedKm * 10) / 10, type: 'gel', note: 'Gel + agua' })
  }
}

function savePlan(): void {
  const name = window.prompt('Nombre del plan', `${strategyLabel.value} · ${formatTime(targetTime.value * 60)}`)
  if (!name?.trim()) return
  savedPlans.value.unshift({ id: Date.now(), name: name.trim(), distance: distance.value, targetTime: targetTime.value, strategy: strategy.value, date: new Date().toLocaleDateString('es-ES'), milestones: JSON.parse(JSON.stringify(milestones.value)), paceData: JSON.parse(JSON.stringify(paceData.value)) })
  persistSavedPlans()
  showSavedPlans.value = true
}

function loadPlan(plan: SavedPlan): void {
  distance.value = plan.distance
  targetTime.value = plan.targetTime
  strategy.value = plan.strategy
  milestones.value = JSON.parse(JSON.stringify(plan.milestones))
  showSavedPlans.value = false
}

function deletePlan(id: number): void {
  savedPlans.value = savedPlans.value.filter((plan) => plan.id !== id)
  persistSavedPlans()
}

async function sharePlan(): Promise<void> {
  const summary = `RUNNER'S PACE LAB\n${formatKm(distance.value)} km · ${formattedTargetTime.value}\n${strategyLabel.value} · ${formatPace(averagePace.value)}/km\nKm ${formatKm(distance.value * 0.72)} → ${formatCompactTime(targetTime.value * 60 * 0.72)}\nMeta → ${formatCompactTime(targetTime.value * 60)}`
  if (navigator.share) {
    await navigator.share({ title: "Runner's Pace Lab", text: summary })
  } else {
    await navigator.clipboard.writeText(summary)
    window.alert('Resumen copiado al portapapeles')
  }
}

function printWristband(): void {
  window.print()
}

function persistSavedPlans(): void {
  localStorage.setItem('runners-pace-lab-plans', JSON.stringify(savedPlans.value))
}

function loadSavedPlans(): SavedPlan[] {
  try {
    const stored = localStorage.getItem('runners-pace-lab-plans')
    return stored ? JSON.parse(stored) as SavedPlan[] : []
  } catch {
    return []
  }
}

function svgX(km: number): number {
  return (km / distance.value) * chartWidth
}

function svgY(pace: number): number {
  const paces = paceData.value.map((point) => point.pace)
  const min = Math.min(...paces) - 0.12
  const max = Math.max(...paces) + 0.12
  return 42 + ((pace - min) / (max - min || 1)) * (chartHeight - 78)
}

function pacePath(): string {
  return paceData.value.map((point, index) => `${index === 0 ? 'M' : 'L'} ${svgX(point.km).toFixed(1)} ${svgY(point.pace).toFixed(1)}`).join(' ')
}

function paceAreaPath(): string {
  return `${pacePath()} L ${chartWidth} ${chartHeight - 20} L 0 ${chartHeight - 20} Z`
}

function elevationY(value: number): number {
  const min = Math.min(...elevationProfile.value) - 10
  const max = Math.max(...elevationProfile.value) + 10
  return 24 + (1 - (value - min) / (max - min || 1)) * (elevationHeight - 46)
}

function elevationPath(): string {
  return elevationProfile.value.map((value, index) => `${index === 0 ? 'M' : 'L'} ${(index / (elevationProfile.value.length - 1) * chartWidth).toFixed(1)} ${elevationY(value).toFixed(1)}`).join(' ')
}

function elevationAreaPath(): string {
  return `${elevationPath()} L ${chartWidth} ${elevationHeight} L 0 ${elevationHeight} Z`
}
</script>

<template>
  <main class="max-w-[1180px] mx-auto px-4 sm:px-7 pb-16">
    <!-- Header -->
    <header class="no-print h-16 sm:h-20 flex justify-between items-center border-b border-teal-100/60">
      <div class="flex gap-2.5 items-center font-display text-xs sm:text-[15px] tracking-wide text-slate-700">
        <span class="w-8 h-8 grid place-items-center rounded-full bg-teal-500 text-white text-xl font-bold -rotate-12">↗</span>
        <span>RUNNER'S <strong class="text-teal-600 font-bold">PACE LAB</strong></span>
      </div>
      <button class="w-9 h-9 sm:w-10 sm:h-10 border border-teal-100 text-slate-600 bg-white rounded-full text-lg hover:bg-teal-50 transition duration-200" aria-label="Perfil de usuario">●</button>
    </header>

    <!-- Hero -->
    <section class="flex flex-col sm:flex-row sm:justify-between sm:items-end gap-5 pt-10 sm:pt-14 pb-8 sm:pb-10">
      <div>
        <p class="mb-2.5 text-teal-600 text-[10px] font-bold tracking-[0.18em] uppercase">Laboratorio de estrategia · 01</p>
        <h1 class="mb-3 font-display text-4xl sm:text-6xl lg:text-7xl tracking-[-0.065em] leading-[0.95] text-slate-800">CREA TU <span class="text-teal-600">CARRERA</span></h1>
        <p class="text-slate-500 text-sm sm:text-base">Construye tu estrategia de ritmo, nutrición y esfuerzo.</p>
      </div>
      <div class="self-start px-3.5 py-2 text-teal-700 bg-teal-50 rounded-full text-[11px] font-bold whitespace-nowrap flex items-center gap-2">
        <span class="inline-block w-1.5 h-1.5 rounded-full bg-teal-500 shadow-[0_0_0_4px_rgba(20,184,166,0.25)]"></span>
        Plan en tiempo real
      </div>
    </section>

    <!-- Config + Summary grid -->
    <section class="grid grid-cols-1 lg:grid-cols-2 gap-4 lg:gap-5 mb-5">
      <!-- Config Card -->
      <article class="no-print relative p-5 sm:p-7 border border-teal-50/80 rounded-2xl bg-white shadow-[0_10px_34px_rgba(18,62,66,0.05)]">
        <div class="flex justify-between items-start gap-4 mb-6">
          <div>
            <p class="mb-2 text-teal-600 text-[10px] font-bold tracking-[0.18em] uppercase">Configuración</p>
            <h2 class="font-display text-lg sm:text-xl tracking-tight text-slate-800">Configura tu carrera</h2>
          </div>
          <span class="text-slate-400 text-xs whitespace-nowrap">01 / 03</span>
        </div>

        <!-- Distance -->
        <label class="block mb-7" for="distance-range">
          <span class="flex justify-between items-baseline gap-3 text-slate-500 text-[11px] font-bold tracking-[0.12em]">
            <span>DISTANCIA</span>
            <strong class="text-slate-800 font-display text-xl sm:text-2xl tracking-tight">{{ formatKm(distance) }} km</strong>
          </span>
          <span class="block mt-1.5 mb-4 text-slate-400 text-xs">{{ distance === 42.195 ? 'Maratón' : distance === 21.097 ? 'Media maratón' : distance === 10 ? 'Carrera urbana' : distance === 5 ? 'Carrera corta' : 'Distancia personalizada' }}</span>
          <input id="distance-range" v-model.number="distance" type="range" min="1" max="100" step="0.001" aria-label="Distancia en kilómetros" class="w-full">
          <span class="flex justify-between mt-2.5 text-slate-400 text-[11px]">
            <button type="button" class="hover:text-teal-600 transition duration-200" @click="distance = 5">5K</button>
            <button type="button" class="hover:text-teal-600 transition duration-200" @click="distance = 10">10K</button>
            <button type="button" class="hover:text-teal-600 transition duration-200" @click="distance = 21.097">21K</button>
            <button type="button" class="hover:text-teal-600 transition duration-200" @click="distance = 42.195">42K</button>
          </span>
        </label>

        <!-- Target Time -->
        <label class="block mb-7" for="time-range">
          <span class="flex justify-between items-baseline gap-3 text-slate-500 text-[11px] font-bold tracking-[0.12em]">
            <span>TIEMPO OBJETIVO</span>
            <strong class="text-slate-800 font-display text-xl sm:text-2xl tracking-tight">{{ formattedTargetTime }}</strong>
          </span>
          <span class="block mt-1.5 mb-4 text-slate-400 text-xs">Ajusta tu ambición minuto a minuto</span>
          <input id="time-range" v-model.number="targetTime" type="range" min="30" max="720" step="1" aria-label="Tiempo objetivo en minutos" class="w-full">
          <span class="flex justify-between mt-2.5 text-slate-400 text-[11px]">
            <span>30m</span><span>3h</span><span>6h</span><span>12h</span>
          </span>
        </label>

        <!-- Strategy -->
        <div class="mb-6">
          <span class="flex justify-between items-baseline gap-3 text-slate-500 text-[11px] font-bold tracking-[0.12em] mb-3">
            <span>ESTRATEGIA</span>
            <strong class="text-slate-800 font-display text-xl sm:text-2xl tracking-tight">Elige tu enfoque</strong>
          </span>
          <div class="grid grid-cols-3 gap-2">
            <button type="button" :class="strategy === 'constant' ? 'border-slate-800 bg-slate-800 text-white shadow-md' : 'border-teal-100/70 bg-teal-50/30 text-slate-600 hover:border-teal-500'" class="min-h-12 border rounded-xl py-2 px-1 text-[11px] sm:text-xs font-bold leading-tight transition duration-200 ease-out" @click="strategy = 'constant'">Ritmo<br>Constante</button>
            <button type="button" :class="strategy === 'negative' ? 'border-slate-800 bg-slate-800 text-white shadow-md' : 'border-teal-100/70 bg-teal-50/30 text-slate-600 hover:border-teal-500'" class="min-h-12 border rounded-xl py-2 px-1 text-[11px] sm:text-xs font-bold leading-tight transition duration-200 ease-out" @click="strategy = 'negative'">↑ Negative<br>Split</button>
            <button type="button" :class="strategy === 'elevation' ? 'border-slate-800 bg-slate-800 text-white shadow-md' : 'border-teal-100/70 bg-teal-50/30 text-slate-600 hover:border-teal-500'" class="min-h-12 border rounded-xl py-2 px-1 text-[11px] sm:text-xs font-bold leading-tight transition duration-200 ease-out" @click="strategy = 'elevation'">Ajustado por<br>Desnivel</button>
          </div>
        </div>

        <button class="w-full min-h-12 border-0 rounded-xl bg-teal-500 text-white font-bold text-[11px] tracking-[0.06em] transition duration-200 ease-out hover:bg-teal-600 hover:-translate-y-0.5" type="button" @click="generateRacePlan">
          RECALCULAR ESTRATEGIA <span class="ml-2 text-base">→</span>
        </button>
      </article>

      <!-- Summary Card (dark) -->
      <article class="relative p-5 sm:p-7 rounded-2xl bg-[#113744] text-white overflow-hidden shadow-[0_10px_34px_rgba(18,62,66,0.08)]">
        <div class="absolute w-64 h-64 -right-30 -top-40 rounded-full bg-teal-400/25"></div>
        <div class="relative">
          <div class="flex justify-between items-center gap-4 mb-1">
            <div>
              <p class="mb-2 text-teal-300 text-[10px] font-bold tracking-[0.18em] uppercase">Resultado principal</p>
              <h2 class="font-display text-lg sm:text-xl tracking-tight">Resumen de carrera</h2>
            </div>
            <span class="px-3 py-1.5 rounded-full text-[11px] font-bold whitespace-nowrap bg-white/10 text-teal-200">LIVE · {{ formatKm(distance) }}K</span>
          </div>

          <div class="grid grid-cols-2 gap-4 py-5 sm:py-6 border-b border-white/10">
            <div>
              <span class="block text-teal-200/70 text-[10px] font-bold tracking-[0.12em]">RITMO MEDIO</span>
              <strong class="block mt-2 font-display text-3xl sm:text-4xl tracking-[-0.08em] leading-none">{{ formatPace(averagePace) }}</strong>
              <small class="text-teal-200/70 text-[11px]">min / km</small>
            </div>
            <div>
              <span class="block text-teal-200/70 text-[10px] font-bold tracking-[0.12em]">META</span>
              <strong class="block mt-2 font-display text-3xl sm:text-4xl tracking-[-0.08em] leading-none">{{ formattedTargetTime }}</strong>
              <small class="text-teal-200/70 text-[11px]">tiempo estimado</small>
            </div>
          </div>

          <div class="flex justify-between items-center gap-3 py-4 border-b border-white/10">
            <div><span class="block text-teal-200/70 text-[10px] font-bold tracking-[0.12em]">ESTRATEGIA</span><strong class="block mt-1 text-[13px]">{{ strategyLabel }}</strong></div>
            <div><span class="block text-teal-200/70 text-[10px] font-bold tracking-[0.12em]">DISTANCIA</span><strong class="block mt-1 text-[13px]">{{ formatKm(distance) }} km</strong></div>
            <div><span class="block text-teal-200/70 text-[10px] font-bold tracking-[0.12em]">DIFICULTAD</span><strong class="block mt-1 text-[13px]">{{ difficulty }}</strong></div>
          </div>

          <div class="py-3">
            <div class="flex justify-between items-center text-teal-100/80 text-xs mb-1">
              <span>Ritmo por kilómetro</span>
              <span class="text-teal-300 text-[10px]">{{ strategy === 'negative' ? 'Final progresivo' : 'Perfil calculado' }}</span>
            </div>
            <svg viewBox="0 0 500 105" role="img" aria-label="Vista previa del ritmo" class="w-full h-[80px] overflow-visible">
              <defs>
                <linearGradient id="miniGradient" x1="0" x2="1">
                  <stop stop-color="#155e75" /><stop offset=".52" stop-color="#13b8ac" /><stop offset="1" stop-color="#f59e0b" />
                </linearGradient>
              </defs>
              <path :d="paceAreaPath()" transform="scale(.555 .362)" fill="url(#miniGradient)" opacity=".18" />
              <path :d="pacePath()" transform="scale(.555 .362)" fill="none" stroke="url(#miniGradient)" stroke-width="6" vector-effect="non-scaling-stroke" />
            </svg>
          </div>

          <div class="flex justify-between items-center pt-4 border-t border-white/10 text-[11px]">
            <span class="flex items-center gap-2 text-teal-200/70">
              <span class="inline-block w-1.5 h-1.5 rounded-full bg-teal-400 shadow-[0_0_0_4px_rgba(20,184,166,0.2)]"></span>
              Estrategia lista
            </span>
            <strong class="text-teal-300 font-bold">Índice {{ strategyScore }}/100</strong>
          </div>
        </div>
      </article>
    </section>

    <!-- Pace/Effort Chart -->
    <section class="relative p-5 sm:p-7 border border-teal-50/80 rounded-2xl bg-white shadow-[0_10px_34px_rgba(18,62,66,0.05)] mb-5">
      <div class="flex flex-col sm:flex-row sm:justify-between sm:items-start gap-3 mb-5">
        <div>
          <p class="mb-2 text-teal-600 text-[10px] font-bold tracking-[0.18em] uppercase">Visualización</p>
          <h2 class="font-display text-lg sm:text-xl tracking-tight text-slate-800">Ritmo / esfuerzo</h2>
          <p class="text-slate-400 text-xs mt-1">Haz click en la línea para añadir un punto táctico.</p>
        </div>
        <div class="flex flex-wrap gap-2.5 text-slate-500 text-[10px]">
          <span class="flex items-center gap-1"><span class="inline-block w-1.5 h-1.5 rounded-full bg-blue-500"></span>Cómodo</span>
          <span class="flex items-center gap-1"><span class="inline-block w-1.5 h-1.5 rounded-full bg-teal-500"></span>Controlado</span>
          <span class="flex items-center gap-1"><span class="inline-block w-1.5 h-1.5 rounded-full bg-yellow-400"></span>Moderado</span>
          <span class="flex items-center gap-1"><span class="inline-block w-1.5 h-1.5 rounded-full bg-orange-500"></span>Exigente</span>
        </div>
      </div>

      <div class="overflow-x-auto">
        <svg class="block w-full min-w-[630px] overflow-visible" viewBox="0 0 900 290" role="img" aria-label="Gráfica interactiva de ritmo y esfuerzo" @click="handleChartClick">
          <defs>
            <linearGradient id="paceGradient" x1="0" y1="0" x2="1" y2="0">
              <stop stop-color="#2563eb" /><stop offset=".35" stop-color="#10b981" /><stop offset=".62" stop-color="#facc15" /><stop offset=".84" stop-color="#f97316" /><stop offset="1" stop-color="#dc2626" />
            </linearGradient>
            <linearGradient id="paceFill" x1="0" y1="0" x2="0" y2="1">
              <stop stop-color="#16b5a7" stop-opacity=".3" /><stop offset="1" stop-color="#fff" stop-opacity="0" />
            </linearGradient>
          </defs>
          <g>
            <line v-for="level in 4" :key="level" x1="0" :y1="level * 56" x2="900" :y2="level * 56" stroke="#e4efee" stroke-width="1" />
          </g>
          <path :d="paceAreaPath()" fill="url(#paceFill)" />
          <path :d="pacePath()" fill="none" stroke="url(#paceGradient)" stroke-width="4" stroke-linecap="round" stroke-linejoin="round" />
          <g v-for="checkpoint in checkpoints" :key="checkpoint" :transform="`translate(${svgX(checkpoint)}, ${svgY(closestPoint(checkpoint).pace)})`">
            <circle r="15" fill="#123b4a" stroke="white" stroke-width="2" />
            <text y="4" text-anchor="middle" fill="white" font-size="7" font-weight="700">{{ checkpoint === distance ? 'META' : `KM ${formatKm(checkpoint)}` }}</text>
          </g>
          <g v-for="milestone in milestones" :key="milestone.id" :transform="`translate(${svgX(milestone.km)}, ${svgY(closestPoint(milestone.km).pace) - 27})`">
            <path d="M0-14 C-11-14-14-5 0 12 C14-5 11-14 0-14Z" fill="#12aa9e" stroke="white" stroke-width="2" />
            <text y="-2" text-anchor="middle" fill="white" font-size="6" font-weight="700">{{ milestoneIcon(milestone.type).slice(0, 2) }}</text>
          </g>
          <line v-if="selectedKilometer !== null" :x1="svgX(selectedKilometer)" y1="0" :x2="svgX(selectedKilometer)" y2="270" stroke="#0c9d93" stroke-width="2" stroke-dasharray="5 4" />
        </svg>
        <div class="flex justify-between min-w-[630px] pt-2 text-slate-400 text-[10px]">
          <span>0 km</span><span>10 km</span><span>{{ formatKm(distance / 2) }} km</span><span>{{ formatKm(distance) }} km</span>
        </div>
      </div>

      <!-- Milestone popup menu -->
      <div v-if="showMilestoneMenu && selectedKilometer !== null" class="max-w-[410px] mx-auto mt-4 p-4 border border-teal-100 rounded-2xl bg-teal-50/30 shadow-lg">
        <div class="flex justify-between items-center mb-3 text-slate-800">
          <strong class="font-display">KM {{ formatKm(selectedKilometer) }}</strong>
          <button type="button" class="text-slate-400 text-xl hover:text-slate-600" @click="showMilestoneMenu = false">×</button>
        </div>
        <div class="grid grid-cols-3 sm:grid-cols-3 gap-2 mb-3">
          <button v-for="type in (['gel', 'water', 'uphill', 'crowd', 'pace', 'note'] as MilestoneType[])" :key="type" type="button" class="p-2 border border-teal-100 rounded-lg bg-white text-slate-600 text-[10px] hover:border-teal-500 hover:text-teal-600 transition duration-200" @click="addMilestone(type)">
            <span class="block mb-1 text-teal-600 text-[9px] font-extrabold">{{ milestoneIcon(type) }}</span>
            {{ milestoneLabel(type) }}
          </button>
        </div>
        <input v-if="selectedMilestoneType === 'note'" v-model="milestoneNote" placeholder="Escribe una nota" aria-label="Nota del hito" class="w-full p-2.5 mb-2.5 border border-teal-100 rounded-lg bg-white text-slate-700 text-xs outline-teal-500">
        <button class="w-full min-h-12 rounded-xl bg-teal-500 text-white font-bold text-[11px] tracking-[0.06em] transition duration-200 hover:bg-teal-600" type="button" @click="addMilestone(selectedMilestoneType)">Añadir hito</button>
      </div>
    </section>

    <!-- Elevation Chart -->
    <section class="relative p-5 sm:p-7 border border-teal-50/80 rounded-2xl bg-white shadow-[0_10px_34px_rgba(18,62,66,0.05)] mb-5">
      <div class="flex flex-col sm:flex-row sm:justify-between sm:items-start gap-3 mb-4">
        <div>
          <p class="mb-2 text-teal-600 text-[10px] font-bold tracking-[0.18em] uppercase">Terreno</p>
          <h2 class="font-display text-lg sm:text-xl tracking-tight text-slate-800">Altimetría</h2>
          <p class="text-slate-400 text-xs mt-1">Perfil ficticio coherente con una estrategia de esfuerzo constante.</p>
        </div>
        <span class="self-start px-3 py-1.5 text-teal-700 bg-teal-50 rounded-full text-[11px] font-bold whitespace-nowrap">+{{ Math.max(...elevationProfile) - Math.min(...elevationProfile) }} m rango</span>
      </div>
      <div class="overflow-x-auto">
        <svg class="block w-full min-w-[630px] overflow-visible" viewBox="0 0 900 170" role="img" aria-label="Perfil de altimetría">
          <defs>
            <linearGradient id="elevationFill" x1="0" y1="0" x2="0" y2="1">
              <stop stop-color="#0f766e" stop-opacity=".28" /><stop offset="1" stop-color="#0f766e" stop-opacity=".02" />
            </linearGradient>
          </defs>
          <path :d="elevationAreaPath()" fill="url(#elevationFill)" />
          <path :d="elevationPath()" fill="none" stroke="#159c91" stroke-width="3" />
          <g v-for="checkpoint in checkpoints" :key="`e-${checkpoint}`">
            <line :x1="svgX(checkpoint)" y1="10" :x2="svgX(checkpoint)" y2="150" stroke="#d4e4e2" stroke-dasharray="3 5" />
            <text :x="svgX(checkpoint)" y="16" text-anchor="middle" fill="#78a09f" font-size="9" font-weight="700">{{ checkpoint === distance ? 'META' : `KM ${formatKm(checkpoint)}` }}</text>
          </g>
        </svg>
      </div>
    </section>

    <!-- Milestones + Metrics grid -->
    <section class="grid grid-cols-1 lg:grid-cols-2 gap-4 lg:gap-5 mb-5">
      <!-- Milestones -->
      <article class="relative p-5 sm:p-7 border border-teal-50/80 rounded-2xl bg-white shadow-[0_10px_34px_rgba(18,62,66,0.05)]">
        <div class="flex justify-between items-start gap-4 mb-5">
          <div>
            <p class="mb-2 text-teal-600 text-[10px] font-bold tracking-[0.18em] uppercase">Táctica</p>
            <h2 class="font-display text-lg sm:text-xl tracking-tight text-slate-800">Hitos de carrera</h2>
          </div>
          <span class="px-3 py-1.5 text-teal-700 bg-teal-50 rounded-full text-[11px] font-bold whitespace-nowrap">{{ milestones.length }} hitos</span>
        </div>
        <div class="grid gap-2 max-h-80 overflow-y-auto mb-4">
          <div v-for="milestone in milestones.slice().sort((a, b) => a.km - b.km)" :key="milestone.id" class="flex items-center gap-3 p-2.5 border border-slate-100 rounded-xl">
            <span class="w-9 h-7 grid place-items-center rounded-lg bg-teal-50 text-teal-600 text-[9px] font-extrabold">{{ milestoneIcon(milestone.type) }}</span>
            <div class="flex-1 grid gap-0.5">
              <strong class="text-slate-800 font-display text-xs">KM {{ formatKm(milestone.km) }}</strong>
              <span class="text-slate-500 text-[11px]">{{ milestoneLabel(milestone.type) }}<template v-if="milestone.note"> · {{ milestone.note }}</template></span>
            </div>
            <button type="button" class="text-slate-400 text-xl hover:text-red-500 transition duration-200" aria-label="Eliminar hito" @click="removeMilestone(milestone.id)">×</button>
          </div>
        </div>
        <button class="w-full min-h-12 border border-teal-100/70 rounded-xl bg-teal-50/30 text-slate-600 font-bold text-[11px] transition duration-200 hover:border-teal-500 hover:-translate-y-0.5" type="button" @click="generateNutritionPlan">
          GENERAR PLAN DE NUTRICIÓN <span class="ml-2 text-base">+</span>
        </button>
      </article>

      <!-- Metrics -->
      <article class="relative p-5 sm:p-7 border border-teal-50/80 rounded-2xl bg-white shadow-[0_10px_34px_rgba(18,62,66,0.05)]">
        <div class="mb-5">
          <p class="mb-2 text-teal-600 text-[10px] font-bold tracking-[0.18em] uppercase">Lecturas</p>
          <h2 class="font-display text-lg sm:text-xl tracking-tight text-slate-800">Métricas adicionales</h2>
        </div>
        <div class="grid grid-cols-2 gap-3">
          <div class="p-3 rounded-xl bg-slate-50">
            <span class="block mb-2 text-slate-400 text-[11px]">Ritmo final</span>
            <strong class="text-slate-800 font-display text-xl">{{ formatPace(finalPace) }}<small class="ml-1 text-slate-400 font-sans text-[10px]">/km</small></strong>
          </div>
          <div class="p-3 rounded-xl bg-slate-50">
            <span class="block mb-2 text-slate-400 text-[11px]">Distancia restante</span>
            <strong class="text-slate-800 font-display text-xl">{{ formatKm(remainingDistance) }}<small class="ml-1 text-slate-400 font-sans text-[10px]"> km</small></strong>
          </div>
          <div class="p-3 rounded-xl bg-slate-50">
            <span class="block mb-2 text-slate-400 text-[11px]">Número de geles</span>
            <strong class="text-slate-800 font-display text-xl">{{ nutritionPlan.length }}</strong>
          </div>
          <div class="p-3 rounded-xl bg-slate-50">
            <span class="block mb-2 text-slate-400 text-[11px]">Índice de estrategia</span>
            <strong class="text-teal-600 font-display text-xl">{{ strategyScore }}<small class="ml-1 text-slate-400 font-sans text-[10px]">/100</small></strong>
          </div>
        </div>
        <div class="flex flex-wrap gap-2 mt-4">
          <span v-if="strategyScore > 75" class="px-2 py-1.5 rounded-full bg-teal-50 text-teal-600 text-[10px] font-bold">Plan equilibrado</span>
          <span v-if="strategy === 'negative'" class="px-2 py-1.5 rounded-full bg-teal-50 text-teal-600 text-[10px] font-bold">Negative split activado</span>
          <span v-if="strategy === 'elevation'" class="px-2 py-1.5 rounded-full bg-teal-50 text-teal-600 text-[10px] font-bold">Desnivel controlado</span>
          <span v-if="nutritionPlan.length >= 2" class="px-2 py-1.5 rounded-full bg-teal-50 text-teal-600 text-[10px] font-bold">Nutrición optimizada</span>
        </div>
      </article>
    </section>

    <!-- Smart Alerts -->
    <section v-if="alerts.length" class="no-print grid gap-2 mb-5">
      <div v-for="alert in alerts" :key="alert" class="flex items-center p-3 border border-amber-200 rounded-xl bg-amber-50 text-amber-700 text-xs">
        <span class="inline-grid w-5 h-5 place-items-center mr-2 rounded-full bg-amber-400 text-white font-bold shrink-0">!</span>
        {{ alert }}
      </div>
    </section>

    <!-- Plan de Pasos Table -->
    <section class="relative p-5 sm:p-7 border border-teal-50/80 rounded-2xl bg-white shadow-[0_10px_34px_rgba(18,62,66,0.05)] mb-5">
      <div class="flex flex-col sm:flex-row sm:justify-between sm:items-start gap-3 mb-5">
        <div>
          <p class="mb-2 text-teal-600 text-[10px] font-bold tracking-[0.18em] uppercase">Ejecución</p>
          <h2 class="font-display text-lg sm:text-xl tracking-tight text-slate-800">Plan de pasos</h2>
          <p class="text-slate-400 text-xs mt-1">Una guía limpia para llevar en carrera.</p>
        </div>
        <div class="no-print flex gap-1 p-1 rounded-lg bg-slate-50 w-full sm:w-auto">
          <button type="button" :class="tableMode === '1' ? 'bg-white text-slate-800 shadow-sm' : 'text-slate-400'" class="flex-1 sm:flex-initial border-0 px-2.5 py-2 rounded-md text-[10px] font-bold transition duration-200" @click="tableMode = '1'">Cada 1 km</button>
          <button type="button" :class="tableMode === '5' ? 'bg-white text-slate-800 shadow-sm' : 'text-slate-400'" class="flex-1 sm:flex-initial border-0 px-2.5 py-2 rounded-md text-[10px] font-bold transition duration-200" @click="tableMode = '5'">Cada 5 km</button>
          <button type="button" :class="tableMode === 'checkpoints' ? 'bg-white text-slate-800 shadow-sm' : 'text-slate-400'" class="flex-1 sm:flex-initial border-0 px-2.5 py-2 rounded-md text-[10px] font-bold transition duration-200" @click="tableMode = 'checkpoints'">Controles</button>
        </div>
      </div>
      <div class="overflow-x-auto">
        <table class="w-full border-collapse min-w-[620px]">
          <thead>
            <tr>
              <th class="py-2.5 px-2 border-b border-teal-50 text-slate-400 text-[10px] tracking-[0.12em] text-left font-bold">KM</th>
              <th class="py-2.5 px-2 border-b border-teal-50 text-slate-400 text-[10px] tracking-[0.12em] text-left font-bold">RITMO</th>
              <th class="py-2.5 px-2 border-b border-teal-50 text-slate-400 text-[10px] tracking-[0.12em] text-left font-bold">ACUMULADO</th>
              <th class="py-2.5 px-2 border-b border-teal-50 text-slate-400 text-[10px] tracking-[0.12em] text-left font-bold">ESFUERZO</th>
              <th class="py-2.5 px-2 border-b border-teal-50 text-slate-400 text-[10px] tracking-[0.12em] text-left font-bold">HITO</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="row in tableRows" :key="row.km" class="border-b border-slate-50">
              <td class="py-3 px-2 text-slate-700 text-xs"><strong class="text-slate-800 font-display">{{ formatKm(row.km) }}</strong></td>
              <td class="py-3 px-2 text-slate-500 text-xs">{{ formatPace(row.pace) }}</td>
              <td class="py-3 px-2 text-slate-500 text-xs">{{ formatCompactTime(row.elapsed * 60) }}</td>
              <td class="py-3 px-2"><span class="px-2 py-1 rounded-full text-[10px] font-bold" :class="effortPillClass(row.effort)">{{ row.effort }}</span></td>
              <td class="py-3 px-2">
                <span v-for="item in milestones.filter((milestone) => Math.abs(milestone.km - row.km) < 0.75)" :key="item.id" class="inline-block mr-1 text-teal-600 text-[10px] font-extrabold">{{ milestoneIcon(item.type) }}</span>
                <span v-if="!milestones.some((milestone) => Math.abs(milestone.km - row.km) < 0.75)" class="text-slate-300 text-xs">—</span>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </section>

    <!-- Action buttons -->
    <section class="no-print grid grid-cols-2 sm:grid-cols-4 gap-2.5 my-5">
      <button class="min-h-12 border border-teal-100/70 rounded-xl bg-teal-50/30 text-slate-600 font-bold text-[11px] transition duration-200 hover:border-teal-500 hover:-translate-y-0.5" type="button" @click="printWristband">IMPRIMIR PULSERA <span class="ml-2 text-base">▣</span></button>
      <button class="min-h-12 border border-teal-100/70 rounded-xl bg-teal-50/30 text-slate-600 font-bold text-[11px] transition duration-200 hover:border-teal-500 hover:-translate-y-0.5" type="button" @click="savePlan">GUARDAR PLAN <span class="ml-2 text-base">★</span></button>
      <button class="min-h-12 border border-teal-100/70 rounded-xl bg-teal-50/30 text-slate-600 font-bold text-[11px] transition duration-200 hover:border-teal-500 hover:-translate-y-0.5" type="button" @click="showComparison = !showComparison">COMPARAR PLANES <span class="ml-2 text-base">⇄</span></button>
      <button class="min-h-12 border-0 rounded-xl bg-teal-500 text-white font-bold text-[11px] tracking-[0.06em] transition duration-200 hover:bg-teal-600 hover:-translate-y-0.5" type="button" @click="sharePlan">COMPARTIR <span class="ml-2 text-base">↗</span></button>
    </section>

    <!-- Saved Plans (Garage) -->
    <section class="no-print relative p-5 sm:p-7 border border-teal-50/80 rounded-2xl bg-white shadow-[0_10px_34px_rgba(18,62,66,0.05)] mb-5">
      <div class="flex justify-between items-start gap-4 mb-5">
        <div>
          <p class="mb-2 text-teal-600 text-[10px] font-bold tracking-[0.18em] uppercase">Archivo local</p>
          <h2 class="font-display text-lg sm:text-xl tracking-tight text-slate-800">Mi garaje</h2>
        </div>
        <button class="text-teal-600 text-[11px] font-bold hover:text-teal-700" type="button" @click="showSavedPlans = !showSavedPlans">{{ showSavedPlans ? 'Ocultar planes' : `${savedPlans.length} planes guardados` }}</button>
      </div>
      <div v-if="showSavedPlans" class="grid gap-2">
        <div v-if="!savedPlans.length" class="p-4 rounded-xl bg-slate-50 text-slate-400 text-xs text-center">Todavía no hay planes guardados. Crea una versión para verla aquí.</div>
        <div v-for="plan in savedPlans" :key="plan.id" class="flex items-center gap-3 p-2.5 border border-slate-100 rounded-xl">
          <div class="flex-1 grid gap-0.5">
            <strong class="text-slate-800 text-[13px]">{{ plan.name }}</strong>
            <span class="text-slate-400 text-[11px]">{{ formatKm(plan.distance) }} km · {{ formatTime(plan.targetTime * 60) }} · {{ plan.date }}</span>
          </div>
          <div class="flex items-center gap-2">
            <button type="button" class="text-teal-600 text-[11px] font-bold hover:text-teal-700" @click="loadPlan(plan)">Cargar</button>
            <button type="button" class="text-slate-400 text-xl hover:text-red-500 transition duration-200" @click="deletePlan(plan.id)">×</button>
          </div>
        </div>
      </div>
    </section>

    <!-- Plan Comparison -->
    <section v-if="showComparison" class="no-print relative p-5 sm:p-7 border border-teal-50/80 rounded-2xl bg-white shadow-[0_10px_34px_rgba(18,62,66,0.05)] mb-5">
      <div class="mb-5">
        <p class="mb-2 text-teal-600 text-[10px] font-bold tracking-[0.18em] uppercase">Análisis</p>
        <h2 class="font-display text-lg sm:text-xl tracking-tight text-slate-800">Comparador de planes</h2>
      </div>
      <div v-if="savedPlans.length < 2" class="p-4 rounded-xl bg-slate-50 text-slate-400 text-xs text-center">Guarda al menos dos planes para compararlos lado a lado.</div>
      <div v-else>
        <div class="grid grid-cols-1 sm:grid-cols-[1fr_auto_1fr] items-center gap-2.5">
          <select v-model.number="compareA" aria-label="Primer plan" class="w-full p-2.5 border border-teal-100 rounded-lg bg-white text-slate-700 text-xs outline-teal-500">
            <option :value="null">Selecciona plan A</option>
            <option v-for="plan in savedPlans" :key="plan.id" :value="plan.id">{{ plan.name }}</option>
          </select>
          <span class="hidden sm:block text-teal-600 font-display font-bold text-center">vs</span>
          <select v-model.number="compareB" aria-label="Segundo plan" class="w-full p-2.5 border border-teal-100 rounded-lg bg-white text-slate-700 text-xs outline-teal-500">
            <option :value="null">Selecciona plan B</option>
            <option v-for="plan in savedPlans" :key="plan.id" :value="plan.id">{{ plan.name }}</option>
          </select>
        </div>
        <div v-if="comparePlans" class="grid grid-cols-1 sm:grid-cols-2 gap-3 mt-4">
          <div v-for="plan in [comparePlans.first, comparePlans.second]" :key="plan.id" class="p-4 rounded-xl bg-slate-50">
            <h3 class="font-display text-base text-slate-800 mb-3">{{ plan.name }}</h3>
            <p class="flex justify-between mb-2.5 text-slate-400 text-[11px]">Tiempo objetivo <strong class="text-slate-700">{{ formatTime(plan.targetTime * 60) }}</strong></p>
            <p class="flex justify-between mb-2.5 text-slate-400 text-[11px]">Ritmo medio <strong class="text-slate-700">{{ formatPace(plan.targetTime / plan.distance) }}/km</strong></p>
            <p class="flex justify-between mb-2.5 text-slate-400 text-[11px]">Estrategia <strong class="text-slate-700">{{ { constant: 'Constante', negative: 'Negative Split', elevation: 'Desnivel' }[plan.strategy] }}</strong></p>
            <p class="flex justify-between mb-2.5 text-slate-400 text-[11px]">Primeros 10 km <strong class="text-slate-700">{{ formatPace(plan.paceData[Math.min(9, plan.paceData.length - 1)].pace) }}/km</strong></p>
            <p class="flex justify-between mb-2.5 text-slate-400 text-[11px]">Últimos 10 km <strong class="text-slate-700">{{ formatPace(plan.paceData[plan.paceData.length - 1].pace) }}/km</strong></p>
            <p class="flex justify-between text-slate-400 text-[11px]">Geles <strong class="text-slate-700">{{ plan.milestones.filter((item) => item.type === 'gel').length }}</strong></p>
          </div>
        </div>
      </div>
    </section>

    <!-- Print-only wristband -->
    <section class="print-band hidden">
      <div class="font-display text-lg font-bold tracking-wide">RUNNER'S PACE LAB</div>
      <div class="flex justify-between mt-4 mb-3 text-base">
        <span>{{ formatKm(distance) }} KM</span>
        <strong>{{ formattedTargetTime }}</strong>
      </div>
      <table class="w-full border-collapse">
        <thead>
          <tr><th class="border border-slate-300 p-1 text-[9px] text-left">KM</th><th class="border border-slate-300 p-1 text-[9px] text-left">TIEMPO</th><th class="border border-slate-300 p-1 text-[9px] text-left">RITMO</th><th class="border border-slate-300 p-1 text-[9px] text-left">HITO</th></tr>
        </thead>
        <tbody>
          <tr v-for="row in paceData" :key="`print-${row.km}`">
            <td class="border border-slate-300 p-1 text-[9px]">{{ formatKm(row.km) }}</td>
            <td class="border border-slate-300 p-1 text-[9px]">{{ formatCompactTime(row.elapsed * 60) }}</td>
            <td class="border border-slate-300 p-1 text-[9px]">{{ formatPace(row.pace) }}</td>
            <td class="border border-slate-300 p-1 text-[9px]">{{ milestones.filter((milestone) => Math.abs(milestone.km - row.km) < 0.75).map((item) => milestoneIcon(item.type)).join(' · ') }}</td>
          </tr>
        </tbody>
      </table>
    </section>
  </main>
</template>
