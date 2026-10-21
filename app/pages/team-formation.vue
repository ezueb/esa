<script setup lang="ts">
  import { parseCsv, validateBidRows, type BidRowError } from '~/utils/bidCsv'
  import { downloadCsv } from '~/utils/csvExport'
  import { titleCase } from '~/utils/labels'
  import { MEETING_DAY_OPTIONS } from '~/utils/options'
  import type { ProjectRead } from '#server/services/projectService'
  import type { StudentRead } from '#server/services/studentService'

  type MeetingDay = 'WEDNESDAY' | 'THURSDAY'

  // ── Semester & meeting day ───────────────────────────────────────────────────

  const semesterId = ref<string | undefined>()
  const meetingDay = ref<MeetingDay>('WEDNESDAY')

  const { semesters } = useSemesters()
  const selectedSemesterLabel = computed(() => {
    const semester = semesters.value.find((s) => s.id === semesterId.value)
    if (!semester) return 'semester'
    const season = semester.season[0] + semester.season.slice(1).toLowerCase()
    return `${season}-${semester.year}`
  })

  const {
    data: projectsForSemester,
    status: projectsStatus,
    refresh: refreshProjectsForSemester,
  } = useFetch<ProjectRead[]>('/api/projects', {
    query: computed(() => ({ semesterId: semesterId.value })),
    watch: [semesterId],
    default: () => [],
  })

  const teamsForDay = computed(() =>
    projectsForSemester.value.flatMap((project) =>
      project.Teams.filter(
        (t) => t.semesterId === semesterId.value && t.meetingDay === meetingDay.value
      ).map((team) => ({
        id: team.id,
        projectId: project.id,
        projectName: project.name,
        partnerName: project.Partner.name,
        memberCount: team.Memberships.length,
      }))
    )
  )

  // Single source of truth for "is this team locked" — reads straight off
  // projectsForSemester (always loaded, not dependent on having clicked Generate
  // Teams this page load) so lock status and the Lock/Unlock button are visible
  // everywhere, not just inside a freshly-generated results view.
  function membershipsForTeam(teamId: string) {
    for (const project of projectsForSemester.value) {
      const team = project.Teams.find((t) => t.id === teamId)
      if (team) return team.Memberships.filter((m) => !m.isMentor)
    }
    return []
  }

  function isTeamLocked(teamId: string) {
    const memberships = membershipsForTeam(teamId)
    return memberships.length > 0 && memberships.every((m) => m.locked)
  }

  const lockingTeamKey = ref<string | null>(null)
  const teamLockError = ref<string | null>(null)

  async function toggleTeamLock(teamId: string, locked: boolean) {
    lockingTeamKey.value = teamId
    teamLockError.value = null
    try {
      await $fetch('/api/team-formation/lock-team', { method: 'POST', body: { teamId, locked } })
      await Promise.all([refreshProjectsForSemester(), refreshStudentsForSemester()])
    } catch (e: any) {
      teamLockError.value = e?.data?.message ?? e?.message ?? 'Failed to update team lock.'
    } finally {
      lockingTeamKey.value = null
    }
  }

  // Pin is purely visual — a way to flag "I'm considering this one" before
  // committing. It never touches the database or the algorithm on its own; only
  // "Lock All Pinned" below actually locks anything. Remembered per-viewer via
  // localStorage, same as it's just a display preference.
  const pinnedTeamIds = ref<Set<string>>(new Set())
  onMounted(() => {
    try {
      const saved = localStorage.getItem('esa-pinned-teams')
      if (saved) pinnedTeamIds.value = new Set(JSON.parse(saved))
    } catch {
      // localStorage unavailable — just start with nothing pinned.
    }
  })
  function persistPins() {
    try {
      localStorage.setItem('esa-pinned-teams', JSON.stringify([...pinnedTeamIds.value]))
    } catch {
      // ignore — nothing to persist to
    }
  }
  function isPinned(teamId: string) {
    return pinnedTeamIds.value.has(teamId)
  }
  function togglePin(teamId: string) {
    const next = new Set(pinnedTeamIds.value)
    if (next.has(teamId)) next.delete(teamId)
    else next.add(teamId)
    pinnedTeamIds.value = next
    persistPins()
  }

  const lockingAllPinned = ref(false)
  async function lockAllPinned() {
    lockingAllPinned.value = true
    teamLockError.value = null
    try {
      for (const teamId of pinnedTeamIds.value) {
        if (!isTeamLocked(teamId)) {
          await $fetch('/api/team-formation/lock-team', { method: 'POST', body: { teamId, locked: true } })
        }
      }
      pinnedTeamIds.value = new Set()
      persistPins()
      await Promise.all([refreshProjectsForSemester(), refreshStudentsForSemester()])
    } catch (e: any) {
      teamLockError.value = e?.data?.message ?? e?.message ?? 'Failed to lock pinned teams.'
    } finally {
      lockingAllPinned.value = false
    }
  }

  const { data: studentsForSemester, refresh: refreshStudentsForSemester } = useFetch<
    StudentRead[]
  >('/api/students', {
    query: computed(() => ({ semesterId: semesterId.value })),
    watch: [semesterId],
    default: () => [],
  })

  const { data: allProjects, refresh: refreshAllProjects } = useFetch<ProjectRead[]>(
    '/api/projects',
    {
      default: () => [],
    }
  )

  const canExport = computed(() => !!semesterId.value && !!meetingDay.value)

  function seasonCode(season: string) {
    if (season === 'SPRING') return 'S'
    if (season === 'SUMMER') return 'U'
    if (season === 'FALL') return 'F'
    return season[0]
  }

  const CHOICE_COLUMN_COUNT = 6

  function exportTeams() {
    const semester = semesters.value.find((s) => s.id === semesterId.value)
    if (!semester) return

    const term = `${seasonCode(semester.season)}${String(semester.year).slice(-2)}`
    const projectLabelById = new Map(
      allProjects.value.map((p) => [p.id, `${term} - ${p.Partner.name}: ${p.name}`])
    )
    const studentById = new Map(studentsForSemester.value.map((s) => [s.id, s]))

    const rows: string[][] = [
      [
        'Team',
        'Student',
        ...Array.from({ length: CHOICE_COLUMN_COUNT }, (_, i) => `Choice ${i + 1}`),
        'Student Major',
        'Student Classification',
        'Gender',
        'Skills',
        'Comments',
      ],
    ]

    for (const project of projectsForSemester.value) {
      const teamLabel =
        projectLabelById.get(project.id) ?? `${term} - ${project.Partner.name}: ${project.name}`

      for (const team of project.Teams) {
        if (team.meetingDay !== meetingDay.value) continue

        for (const membership of team.Memberships) {
          const student = studentById.get(membership.studentId)
          const enrollment = student?.Enrollments.find((e) => e.semesterId === semesterId.value)

          const choices = (student?.Choices ?? [])
            .filter((c) => c.semesterId === semesterId.value)
            .sort((a, b) => a.rank - b.rank)
            .map((c) => projectLabelById.get(c.projectId) ?? '')
          const choiceCells = Array.from(
            { length: CHOICE_COLUMN_COUNT },
            (_, i) => choices[i] ?? ''
          )

          rows.push([
            teamLabel,
            `${membership.Student.lastName}, ${membership.Student.firstName}`,
            ...choiceCells,
            enrollment?.major ?? '',
            enrollment ? titleCase(enrollment.year) : '',
            enrollment ? titleCase(enrollment.gender) : '',
            enrollment?.skills.join('; ') ?? '',
            enrollment?.comments ?? '',
          ])
        }
      }
    }

    const dayLabel = meetingDay.value === 'WEDNESDAY' ? 'Wed' : 'Thu'
    downloadCsv(`${dayLabel} Teams - ${selectedSemesterLabel.value}.csv`, rows)
  }

  // ── Bid file upload ──────────────────────────────────────────────────────────

  const bidFile = ref<File | null>(null)
  const parsedRows = ref<Record<string, string>[]>([])
  const parseErrors = ref<BidRowError[]>([])
  const missingColumns = ref<string[]>([])
  const uploadResult = ref<{
    studentsImported: number
    choicesCreated: number
    skippedStudents: string[]
  } | null>(null)
  const uploading = ref(false)
  const uploadError = ref<string | null>(null)

  watch(bidFile, async (file) => {
    parsedRows.value = []
    parseErrors.value = []
    missingColumns.value = []
    if (!file) return

    // Only clear prior results once a new file is actually chosen, so a successful
    // upload's summary stays visible after we reset `bidFile` below.
    uploadResult.value = null
    uploadError.value = null

    const text = await file.text()
    const rows = parseCsv(text)
    const validation = validateBidRows(rows)
    missingColumns.value = validation.missingColumns
    parseErrors.value = validation.errors
    if (!validation.missingColumns.length && !validation.errors.length) {
      parsedRows.value = rows
    }
  })

  const canUpload = computed(
    () =>
      !!semesterId.value &&
      parsedRows.value.length > 0 &&
      !parseErrors.value.length &&
      !missingColumns.value.length
  )

  async function submitBidFile() {
    if (!canUpload.value || !semesterId.value) return
    uploading.value = true
    uploadError.value = null
    try {
      uploadResult.value = await $fetch('/api/team-formation/bids', {
        method: 'POST',
        query: { semesterId: semesterId.value, meetingDay: meetingDay.value },
        body: parsedRows.value,
      })
      // Require an explicit new file selection before another upload can be submitted —
      // otherwise switching the meeting day and re-clicking Upload silently resubmits the
      // same roster under the new day, overwriting those students' enrollment day.
      bidFile.value = null
      await Promise.all([
        refreshProjectsForSemester(),
        refreshAllProjects(),
        refreshStudentsForSemester(),
      ])
    } catch (e: any) {
      uploadError.value = e?.data?.message ?? e?.message ?? 'Failed to import bid file.'
    } finally {
      uploading.value = false
    }
  }

  // ── Step-by-step page layout ─────────────────────────────────────────────────

  // One step on screen at a time — the others aren't rendered at all. Purely
  // visual, never touches any selection, lock, or data underneath. Staff move
  // freely with the step bar or Back/Next; nothing advances on its own, so a
  // step's results (e.g. an upload summary) stay up until they choose to move on.
  // Every step past the first needs a semester, so those stay disabled until then.
  const activeSection = ref(1)

  const STEP_COUNT = 5
  const steps = computed(() =>
    [
      { title: 'Semester & Day', icon: 'i-heroicons-calendar' },
      { title: 'Bid File', icon: 'i-heroicons-arrow-up-tray' },
      { title: 'Lock 3200s', icon: 'i-heroicons-lock-closed' },
      { title: 'Generate', icon: 'i-heroicons-sparkles' },
      { title: 'Team Board', icon: 'i-heroicons-squares-2x2' },
    ].map((step, i) => ({ ...step, disabled: i > 0 && !semesterId.value }))
  )
  // UStepper treats a numeric v-model as a 0-based index; sections are numbered from 1.
  const stepIndex = computed({
    get: () => activeSection.value - 1,
    set: (i: number) => (activeSection.value = i + 1),
  })

  // ── 3200 lock-in ──────────────────────────────────────────────────────────────

  const students3200ForDay = computed(() =>
    studentsForSemester.value.filter((s) => {
      const enrollment = s.Enrollments.find((e) => e.semesterId === semesterId.value)
      return enrollment?.class === 'EPCS_3200' && enrollment.meetingDay === meetingDay.value
    })
  )

  function lockedMembershipFor(student: StudentRead) {
    return student.Memberships.find(
      (m) =>
        m.locked &&
        m.Team.semesterId === semesterId.value &&
        m.Team.meetingDay === meetingDay.value
    )
  }

  function choiceLabelsFor(student: StudentRead) {
    return student.Choices.filter((c) => c.semesterId === semesterId.value)
      .sort((a, b) => a.rank - b.rank)
      .map((c) => allProjects.value.find((p) => p.id === c.projectId)?.name ?? '')
      .filter(Boolean)
  }

  const SEASON_ORDER: Record<string, number> = { SPRING: 0, SUMMER: 1, FALL: 2 }

  // The project a student most recently worked on in an earlier semester, if any —
  // mirrors the "returning student" logic the solver itself uses, computed here from
  // data already on the student object so the dropdown/labels can show it live.
  function previousProjectFor(student: StudentRead): string | undefined {
    const currentSemester = semesters.value.find((s) => s.id === semesterId.value)
    if (!currentSemester) return undefined
    const currentRank = currentSemester.year * 10 + SEASON_ORDER[currentSemester.season]!

    let best: { rank: number; name: string } | undefined
    for (const m of student.Memberships) {
      if (m.isMentor) continue
      const sem = m.Team.Semester
      const rank = sem.year * 10 + SEASON_ORDER[sem.season]!
      if (rank >= currentRank) continue
      if (!best || rank > best.rank) {
        best = { rank, name: m.Team.Project.name }
      }
    }
    return best?.name
  }

  function ordinal(n: number) {
    if (n % 10 === 1 && n % 100 !== 11) return `${n}st`
    if (n % 10 === 2 && n % 100 !== 12) return `${n}nd`
    if (n % 10 === 3 && n % 100 !== 13) return `${n}rd`
    return `${n}th`
  }

  // Team dropdown options for one student, ordered by THEIR ranking (1st choice on
  // top) and labeled with the rank number, so staff can pick straight off the list
  // instead of cross-referencing the Choices line above. Teams they didn't rank at
  // all sort to the bottom, prefixed with "—" instead of a number. Their returning
  // project (if any) is flagged regardless of where it falls in their ranking.
  function teamOptionsFor(student: StudentRead) {
    const rankByProjectName = new Map(choiceLabelsFor(student).map((name, i) => [name, i + 1]))
    const prevProject = previousProjectFor(student)
    return teamsForDay.value
      .map((t) => ({ team: t, rank: rankByProjectName.get(t.projectName) }))
      .sort((a, b) => (a.rank ?? Infinity) - (b.rank ?? Infinity))
      .map(({ team, rank }) => {
        const base = rank ? `${rank}. ${team.projectName}` : `Not ranked: ${team.projectName}`
        return {
          label: team.projectName === prevProject ? `${base} ⟲ returning` : base,
          value: team.id,
        }
      })
  }

  // Live label for whatever team is CURRENTLY picked in the dropdown — recomputes as
  // staff changes the selection, whether it started from a suggestion or a manual pick.
  function selectionLabel(student: StudentRead) {
    const teamId = lockSelections[student.id]
    if (!teamId) return null

    const suggestion = suggestions[student.id]
    if (suggestion && suggestion.teamId === teamId) {
      return `Suggested: ${suggestion.reason}`
    }

    const projectName = teamsForDay.value.find((t) => t.id === teamId)?.projectName
    if (!projectName) return null
    const isReturning = projectName === previousProjectFor(student)
    const idx = choiceLabelsFor(student).indexOf(projectName)
    const rankPart = idx === -1 ? 'not one of their ranked choices' : `their ${ordinal(idx + 1)} choice`
    return isReturning ? `This is their returning project, and ${rankPart}` : `This is ${rankPart}`
  }

  const lockSelections = reactive<Record<string, string>>({})
  const lockingKey = ref<string | null>(null)
  const lockError = ref<string | null>(null)

  async function lockStudent(studentId: string) {
    const teamId = lockSelections[studentId]
    if (!teamId) return
    lockingKey.value = studentId
    lockError.value = null
    try {
      await $fetch('/api/team-formation/lock', { method: 'POST', body: { teamId, studentId } })
      await refreshStudentsForSemester()
    } catch (e: any) {
      lockError.value = e?.data?.message ?? e?.message ?? 'Failed to lock student.'
    } finally {
      lockingKey.value = null
    }
  }

  async function unlockMembership(membershipId: string) {
    lockingKey.value = membershipId
    lockError.value = null
    try {
      await $fetch('/api/team-formation/lock', { method: 'DELETE', body: { membershipId } })
      await refreshStudentsForSemester()
    } catch (e: any) {
      lockError.value = e?.data?.message ?? e?.message ?? 'Failed to unlock student.'
    } finally {
      lockingKey.value = null
    }
  }

  // Algorithm-suggested placements for unlocked 3200 students, shown as a pre-filled
  // team choice + a reason staff can review before locking anything in.
  interface Suggestion {
    teamId: string
    reason: string
  }
  const suggestions = reactive<Record<string, Suggestion>>({})
  const noChoiceStudentIds = ref<Set<string>>(new Set())
  const suggesting = ref(false)
  const suggestError = ref<string | null>(null)
  const usedFallbackThirdChoice = ref(false)
  const lockingAllSuggested = ref(false)
  // How many 3200 students a single project can absorb before the suggestion step
  // starts pushing the rest to their next choice — keeps a popular project from
  // soaking up every available 3200 and leaving another project without one.
  const maxUpperclassPerProject = ref(4)

  function reasonFor(placement: { choiceRank: number; isReturning: boolean }) {
    if (placement.isReturning) return 'Returning to their previous project'
    if (placement.choiceRank === 1) return 'Their 1st choice'
    if (placement.choiceRank === 2) return 'Their 2nd choice'
    return `Their ${placement.choiceRank}rd choice, since 1st and 2nd were full`
  }

  async function suggestPlacements() {
    if (!semesterId.value) return
    suggesting.value = true
    suggestError.value = null
    try {
      const result = await $fetch<{
        placements: { studentId: string; teamId: string; choiceRank: number; isReturning: boolean }[]
        noChoicesStudentIds: string[]
        usedFallbackThirdChoice: boolean
      }>('/api/team-formation/suggest-locks', {
        method: 'POST',
        body: {
          semesterId: semesterId.value,
          day: meetingDay.value,
          maxTeamSize: config.max_team_size,
          maxUpperclassPerProject: maxUpperclassPerProject.value,
        },
      })
      for (const key of Object.keys(suggestions)) delete suggestions[key]
      for (const p of result.placements) {
        lockSelections[p.studentId] = p.teamId
        suggestions[p.studentId] = { teamId: p.teamId, reason: reasonFor(p) }
      }
      noChoiceStudentIds.value = new Set(result.noChoicesStudentIds)
      usedFallbackThirdChoice.value = result.usedFallbackThirdChoice
    } catch (e: any) {
      suggestError.value = e?.data?.message ?? e?.message ?? 'Failed to compute suggestions.'
    } finally {
      suggesting.value = false
    }
  }

  async function lockAllSuggested() {
    const studentIds = Object.keys(suggestions)
    lockingAllSuggested.value = true
    lockError.value = null
    try {
      for (const studentId of studentIds) {
        await $fetch('/api/team-formation/lock', {
          method: 'POST',
          body: { teamId: suggestions[studentId]!.teamId, studentId },
        })
        delete suggestions[studentId]
      }
      await refreshStudentsForSemester()
    } catch (e: any) {
      lockError.value = e?.data?.message ?? e?.message ?? 'Failed to lock all suggested students.'
    } finally {
      lockingAllSuggested.value = false
    }
  }

  // ── Team generation ──────────────────────────────────────────────────────────

  const config = reactive({
    min_team_size: 4,
    max_team_size: 6,
    prioritize_returning_students: true,
    prioritize_3200_first_choice: true,
    balance_gender: true,
  })

  // Defaults to 1 (every team needs a 3200) per the solver's own default — set to 0
  // yourself if there genuinely aren't enough 3200 students to go around this semester.
  const minUpperclassPerProject = ref(1)

  const generating = ref(false)
  const generateError = ref<string | null>(null)
  interface ResultStudent {
    id: string
    firstName: string
    lastName: string
    netID: string
    Choices: { projectId: string; rank: number }[]
    Enrollments: { semesterId: string; class: string; major: string }[]
  }
  interface GenerateResponse {
    teamAssignments: Record<string, ResultStudent[]>
    projects: { id: string; name: string }[]
    teamMeta: Record<string, { projectId: string; meetingDay: MeetingDay; projectName: string }>
  }
  const generateResult = ref<GenerateResponse | null>(null)

  function is3200(student: ResultStudent) {
    return student.Enrollments.find((e) => e.semesterId === semesterId.value)?.class === 'EPCS_3200'
  }
  function majorFor(student: ResultStudent) {
    return student.Enrollments.find((e) => e.semesterId === semesterId.value)?.major ?? 'Unknown'
  }
  // Off by default so the results view doesn't dump everyone's full ranking in
  // staff's face — flip it on only if you actually want to see choice rank per student.
  const showChoiceRank = ref(false)

  function choiceRankLabel(student: ResultStudent, teamId: string) {
    const projectId = generateResult.value?.teamMeta[teamId]?.projectId
    const choice = student.Choices.find((c) => c.projectId === projectId)
    return choice ? `${ordinal(choice.rank)} choice` : 'not ranked'
  }

  // ── Move a student after results are already in ─────────────────────────────
  // Deliberately free-floating by default: moving someone here does NOT lock them
  // in (unless the staff member checks the box), so it stays as flexible as an
  // unassigned student — a future Generate Teams run can still touch them, or
  // staff can move them again just as easily.

  function resultChoiceLabels(student: ResultStudent) {
    return student.Choices.slice()
      .sort((a, b) => a.rank - b.rank)
      .map((c) => generateResult.value?.projects.find((p) => p.id === c.projectId)?.name ?? '')
      .filter(Boolean)
  }

  function resultTeamOptionsFor(student: ResultStudent) {
    const rankByProjectName = new Map(resultChoiceLabels(student).map((name, i) => [name, i + 1]))
    return Object.entries(generateResult.value?.teamMeta ?? {})
      .map(([teamId, meta]) => ({ teamId, name: meta.projectName, rank: rankByProjectName.get(meta.projectName) }))
      .sort((a, b) => (a.rank ?? Infinity) - (b.rank ?? Infinity))
      .map(({ teamId, name, rank }) => ({
        label: rank ? `${rank}. ${name}` : `Not ranked: ${name}`,
        value: teamId,
      }))
  }

  const openMovePanel = ref<string | null>(null)
  const moveSelections = reactive<Record<string, string>>({})
  const moveLockChecked = reactive<Record<string, boolean>>({})
  const movingKey = ref<string | null>(null)
  const moveError = ref<string | null>(null)

  // Reflect a move in the results view directly — no need to re-run the solver
  // just to show someone in their new team.
  function reflectMoveInResults(studentId: string, fromTeamId: string, toTeamId: string) {
    if (!generateResult.value) return
    const fromList = generateResult.value.teamAssignments[fromTeamId]
    const idx = fromList?.findIndex((s) => s.id === studentId) ?? -1
    if (idx === -1 || !fromList) return
    const [moved] = fromList.splice(idx, 1)
    if (moved) {
      generateResult.value.teamAssignments[toTeamId] ??= []
      generateResult.value.teamAssignments[toTeamId]!.push(moved)
    }
  }

  async function moveStudent(studentId: string, fromTeamId: string) {
    const teamId = moveSelections[studentId]
    if (!teamId || teamId === fromTeamId) return
    movingKey.value = studentId
    moveError.value = null
    try {
      await $fetch('/api/team-formation/lock', {
        method: 'POST',
        body: { teamId, studentId, locked: !!moveLockChecked[studentId] },
      })
      reflectMoveInResults(studentId, fromTeamId, teamId)
      delete moveSelections[studentId]
      openMovePanel.value = null
      await Promise.all([refreshProjectsForSemester(), refreshStudentsForSemester()])
    } catch (e: any) {
      moveError.value = e?.data?.message ?? e?.message ?? 'Failed to move student.'
    } finally {
      movingKey.value = null
    }
  }

  const canGenerate = computed(() => !!semesterId.value && teamsForDay.value.length > 0)

  async function generateTeams() {
    if (!canGenerate.value || !semesterId.value) return
    generating.value = true
    generateError.value = null
    generateResult.value = null
    try {
      generateResult.value = await $fetch<GenerateResponse>('/api/team-formation/generate', {
        method: 'POST',
        body: {
          semesterId: semesterId.value,
          day: meetingDay.value,
          config: {
            ...config,
            max_upperclass_per_project: maxUpperclassPerProject.value,
            min_upperclass_per_project: minUpperclassPerProject.value,
          },
        },
      })
      await Promise.all([
        refreshProjectsForSemester(),
        refreshAllProjects(),
        refreshStudentsForSemester(),
      ])
    } catch (e: any) {
      generateError.value = e?.data?.message ?? e?.message ?? 'Team generation failed.'
    } finally {
      generating.value = false
    }
  }

  function studentsFor(teamId: string) {
    return generateResult.value?.teamAssignments[teamId] ?? []
  }
  // ── Team board ───────────────────────────────────────────────────────────────
  // Drag-and-drop view of every team for the semester/day, split into three
  // columns: Teams (everything not set aside), Workspace (teams set aside to work
  // on together — the same set as Pin, so it's purely visual), and Locked (backed
  // by Membership.locked, so dropping a team there really locks it). Students can
  // be dragged between Workspace teams, with each student's ranking shown so staff
  // can see where they'd actually be happiest. Reads straight from the database
  // rows, so it works with or without a Generate Teams run this page load.

  type BoardZone = 'pool' | 'workspace' | 'locked'
  type BoardDrag =
    | { kind: 'team'; teamId: string }
    | { kind: 'student'; studentId: string; fromTeamId: string }

  const BOARD_ZONES: { zone: BoardZone; title: string; hint: string }[] = [
    { zone: 'pool', title: 'Teams', hint: 'Drag a team to the Workspace to work on it.' },
    {
      zone: 'workspace',
      title: 'Workspace',
      hint: 'Drag students between these teams. Each team shows how the dragged student ranked it.',
    },
    { zone: 'locked', title: 'Locked', hint: 'Drop a team here to lock it. Drag it out to unlock.' },
  ]

  const boardDrag = ref<BoardDrag | null>(null)
  const boardDropTarget = ref<string | null>(null)
  const boardBusy = ref(false)
  const boardError = ref<string | null>(null)
  const expandedBoardStudent = ref<string | null>(null)

  function zoneFor(teamId: string): BoardZone {
    if (isTeamLocked(teamId)) return 'locked'
    if (isPinned(teamId)) return 'workspace'
    return 'pool'
  }

  function boardTeamsIn(zone: BoardZone) {
    return teamsForDay.value.filter((t) => zoneFor(t.id) === zone)
  }

  const studentById = computed(() => new Map(studentsForSemester.value.map((s) => [s.id, s])))

  function boardRank(studentId: string, projectId: string) {
    return studentById.value
      .get(studentId)
      ?.Choices.find((c) => c.semesterId === semesterId.value && c.projectId === projectId)?.rank
  }

  function boardIs3200(studentId: string) {
    const enrollment = studentById.value
      .get(studentId)
      ?.Enrollments.find((e) => e.semesterId === semesterId.value)
    return enrollment?.class === 'EPCS_3200'
  }

  function boardCount3200(teamId: string) {
    return membershipsForTeam(teamId).filter((m) => boardIs3200(m.studentId)).length
  }

  function rankText(rank: number | undefined) {
    return rank ? ordinal(rank) : 'not ranked'
  }

  // Green for 1st/2nd, amber for 3rd, gray for anything lower or unranked.
  function rankColor(rank: number | undefined) {
    if (rank && rank <= 2) return 'success'
    if (rank === 3) return 'warning'
    return 'neutral'
  }

  // The student's full ranking, with Workspace teams flagged so staff can see at a
  // glance which of their preferences are actually on the table right now.
  function boardChoiceList(studentId: string) {
    const student = studentById.value.get(studentId)
    if (!student) return []
    const workspaceProjectIds = new Set(boardTeamsIn('workspace').map((t) => t.projectId))
    return student.Choices.filter((c) => c.semesterId === semesterId.value)
      .sort((a, b) => a.rank - b.rank)
      .map((c) => ({
        rank: c.rank,
        name: allProjects.value.find((p) => p.id === c.projectId)?.name ?? 'Unknown project',
        inWorkspace: workspaceProjectIds.has(c.projectId),
      }))
  }

  function onBoardDragStart(event: DragEvent, drag: BoardDrag) {
    if (boardBusy.value) {
      event.preventDefault()
      return
    }
    boardDrag.value = drag
    boardError.value = null
    if (event.dataTransfer) {
      event.dataTransfer.effectAllowed = 'move'
      // Firefox won't start a drag without some data set.
      event.dataTransfer.setData('text/plain', drag.kind === 'team' ? drag.teamId : drag.studentId)
    }
  }

  function onBoardDragEnd() {
    boardDrag.value = null
    boardDropTarget.value = null
  }

  function canDropTeamOn(zone: BoardZone) {
    const drag = boardDrag.value
    return drag?.kind === 'team' && zoneFor(drag.teamId) !== zone
  }

  function canDropStudentOn(teamId: string) {
    const drag = boardDrag.value
    return (
      drag?.kind === 'student' && drag.fromTeamId !== teamId && zoneFor(teamId) === 'workspace'
    )
  }

  function onZoneDragOver(event: DragEvent, zone: BoardZone) {
    if (!canDropTeamOn(zone)) return
    event.preventDefault()
    boardDropTarget.value = zone
  }

  function onTeamDragOver(event: DragEvent, teamId: string) {
    if (!canDropStudentOn(teamId)) return
    event.preventDefault()
    event.stopPropagation()
    boardDropTarget.value = teamId
  }

  async function onZoneDrop(zone: BoardZone) {
    const drag = boardDrag.value
    onBoardDragEnd()
    if (drag?.kind !== 'team') return
    await moveTeamToZone(drag.teamId, zone)
  }

  async function onTeamDrop(teamId: string) {
    const drag = boardDrag.value
    onBoardDragEnd()
    if (drag?.kind !== 'student' || drag.fromTeamId === teamId) return
    await moveStudentOnBoard(drag.studentId, drag.fromTeamId, teamId)
  }

  async function moveTeamToZone(teamId: string, zone: BoardZone) {
    const from = zoneFor(teamId)
    if (from === zone) return
    if (zone === 'locked' && !membershipsForTeam(teamId).length) {
      boardError.value = "This team has no members yet, so there's nothing to lock."
      return
    }

    boardBusy.value = true
    try {
      if (zone === 'locked' || from === 'locked') {
        await $fetch('/api/team-formation/lock-team', {
          method: 'POST',
          body: { teamId, locked: zone === 'locked' },
        })
      }
      // Workspace membership is just the pin set — keep it in step with the column.
      if (isPinned(teamId) !== (zone === 'workspace')) togglePin(teamId)
      if (zone === 'locked' || from === 'locked') {
        await Promise.all([refreshProjectsForSemester(), refreshStudentsForSemester()])
      }
    } catch (e: any) {
      boardError.value = e?.data?.message ?? e?.message ?? 'Failed to move team.'
    } finally {
      boardBusy.value = false
    }
  }

  async function moveStudentOnBoard(studentId: string, fromTeamId: string, toTeamId: string) {
    // Keep whatever lock state the student already had — a locked 3200 stays locked,
    // just in the new team; everyone else stays free for the next Generate Teams run.
    const membership = membershipsForTeam(fromTeamId).find((m) => m.studentId === studentId)
    boardBusy.value = true
    try {
      await $fetch('/api/team-formation/lock', {
        method: 'POST',
        body: {
          teamId: toTeamId,
          studentId,
          locked: !!membership?.locked,
          replaceLocked: true,
        },
      })
      reflectMoveInResults(studentId, fromTeamId, toTeamId)
      await Promise.all([refreshProjectsForSemester(), refreshStudentsForSemester()])
    } catch (e: any) {
      boardError.value = e?.data?.message ?? e?.message ?? 'Failed to move student.'
    } finally {
      boardBusy.value = false
    }
  }
</script>

<template>
  <UContainer
    :class="['space-y-8 py-8', activeSection === 5 ? 'max-w-7xl' : 'max-w-4xl']"
  >
    <UStepper v-model="stepIndex" :items="steps" :linear="false" size="sm" />
    <section v-if="activeSection === 1" class="space-y-4">
      <h2 class="text-lg font-semibold">1. Semester & Meeting Day</h2>

        <UFormField label="Semester">
          <SemesterFilter v-model="semesterId" />
        </UFormField>

        <UFormField label="Meeting Day">
          <URadioGroup v-model="meetingDay" orientation="horizontal" :items="MEETING_DAY_OPTIONS" />
        </UFormField>

        <UButton
          label="Export Teams"
          icon="i-heroicons-arrow-down-tray"
          color="neutral"
          variant="soft"
          :disabled="!canExport"
          @click="exportTeams"
        />

      <div v-if="semesterId" class="space-y-2">
        <div class="flex items-center justify-between gap-2">
          <h3 class="font-medium">Teams already entered for this semester/day</h3>
          <UButton
            v-if="pinnedTeamIds.size"
            :label="`Lock All Pinned (${pinnedTeamIds.size})`"
            icon="i-heroicons-lock-closed"
            size="xs"
            :loading="lockingAllPinned"
            @click="lockAllPinned"
          />
        </div>
        <UAlert
          v-if="projectsStatus !== 'pending' && !teamsForDay.length"
          icon="i-heroicons-exclamation-triangle"
          color="warning"
          variant="subtle"
          title="No teams found"
          description="Create projects and teams for this semester/day in the Database tab before continuing."
        />
        <UAlert
          v-if="teamLockError"
          color="error"
          variant="subtle"
          title="Could not update team lock"
          :description="teamLockError"
        />

        <ul v-else class="space-y-1">
          <li
            v-for="team in teamsForDay"
            :key="team.id"
            class="flex flex-wrap items-center justify-between gap-2 rounded border border-gray-200 p-2 text-sm dark:border-gray-800"
          >
            <span>
              {{ team.projectName }} ({{ team.partnerName }})
              <UBadge v-if="isTeamLocked(team.id)" color="primary" variant="subtle" size="sm">Locked</UBadge>
              <UBadge v-if="isPinned(team.id)" color="warning" variant="subtle" size="sm">Pinned</UBadge>
            </span>
            <div class="flex items-center gap-2">
              <span class="text-gray-500">{{ team.memberCount }} member(s)</span>
              <UButton
                :icon="isPinned(team.id) ? 'i-heroicons-bookmark-slash' : 'i-heroicons-bookmark'"
                :label="isPinned(team.id) ? 'Unpin' : 'Pin'"
                size="xs"
                color="warning"
                variant="ghost"
                @click="togglePin(team.id)"
              />
              <UButton
                v-if="team.memberCount > 0"
                :label="isTeamLocked(team.id) ? 'Unlock Team' : 'Lock Team'"
                size="xs"
                :color="isTeamLocked(team.id) ? 'neutral' : 'primary'"
                :variant="isTeamLocked(team.id) ? 'soft' : 'solid'"
                :loading="lockingTeamKey === team.id"
                @click="toggleTeamLock(team.id, !isTeamLocked(team.id))"
              />
            </div>
          </li>
        </ul>
      </div>
    </section>

    <section v-if="activeSection === 2" class="space-y-4">
      <h2 class="text-lg font-semibold">2. Bid File</h2>

      <UFormField label="Bid Response CSV">
        <UFileUpload
          v-model="bidFile"
          accept=".csv"
          label="Drop CSV here or click to browse"
          :disabled="!semesterId"
        />
      </UFormField>
      <p v-if="!semesterId" class="text-sm text-gray-500">Select a semester in step 1 first.</p>

      <UAlert
        v-if="missingColumns.length"
        color="error"
        variant="subtle"
        title="Missing required columns"
        :description="missingColumns.join(', ')"
      />

      <div v-else-if="parseErrors.length" class="space-y-2">
        <UAlert
          color="error"
          variant="subtle"
          :title="`${parseErrors.length} error(s) found. Upload aborted.`"
        />
        <ul class="max-h-64 space-y-1 overflow-y-auto text-sm">
          <li v-for="(err, i) in parseErrors" :key="i" class="text-red-600 dark:text-red-400">
            Row {{ err.row }}: {{ err.message }}
          </li>
        </ul>
      </div>

      <UAlert
        v-else-if="parsedRows.length"
        color="success"
        variant="subtle"
        :title="`${parsedRows.length} row(s) validated successfully`"
      />

      <UButton
        label="Upload"
        icon="i-heroicons-arrow-up-tray"
        :loading="uploading"
        :disabled="!canUpload || uploading"
        @click="submitBidFile"
      />

      <UAlert
        v-if="uploadError"
        color="error"
        variant="subtle"
        title="Upload failed"
        :description="uploadError"
      />

      <div v-if="uploadResult" class="space-y-2">
        <UAlert
          color="success"
          variant="subtle"
          title="Import complete"
          :description="`${uploadResult.studentsImported} student(s) imported, ${uploadResult.choicesCreated} choice(s) recorded.`"
        />
        <UAlert
          v-if="uploadResult.skippedStudents.length"
          color="warning"
          variant="subtle"
          title="Skipped rows (missing SSO ID)"
          :description="uploadResult.skippedStudents.join(', ')"
        />
      </div>
    </section>

    <section v-if="activeSection === 3" class="space-y-4">
      <h2 class="text-lg font-semibold">3. Lock 3200 Students</h2>

      <p class="text-sm text-gray-500">
        Place specific 3200 students into a team ahead of time. Locked students are
        skipped by Team Generation (step 4) and kept as is. Everyone else, including any
        3200 students you leave unlocked, is assigned by Team Generation.
      </p>

      <p v-if="!semesterId" class="text-sm text-gray-500">Select a semester in step 1 first.</p>

      <div v-if="semesterId" class="grid grid-cols-2 gap-4">
        <UFormField label="Min 3200 per Project">
          <UInputNumber v-model="minUpperclassPerProject" :min="0" class="w-full" />
        </UFormField>
        <UFormField label="Max 3200 per Project">
          <UInputNumber v-model="maxUpperclassPerProject" :min="minUpperclassPerProject || 1" class="w-full" />
        </UFormField>
      </div>

      <div v-if="semesterId" class="flex flex-wrap items-end gap-2">
        <UButton
          label="Suggest Placements"
          icon="i-heroicons-light-bulb"
          color="neutral"
          variant="soft"
          :loading="suggesting"
          @click="suggestPlacements"
        />
        <UButton
          v-if="Object.keys(suggestions).length"
          label="Lock All Suggested"
          icon="i-heroicons-lock-closed"
          :loading="lockingAllSuggested"
          @click="lockAllSuggested"
        />
      </div>
      <p class="text-sm text-gray-500">
        "Suggest Placements" runs the algorithm on just the unlocked 3200 students,
        guaranteeing each one their 1st or 2nd choice whenever possible, and pre-fills
        the team below with a reason. Review it, then lock individually or all at once.
        Max above also applies when you hit Generate Teams later, so a popular project
        can't absorb all the 3200s and leave another project without one. Min is 1 by
        default (every project gets at least one 3200) and only applies to Generate
        Teams. It's tight by nature, so set it to 0 if there aren't enough 3200
        students to go around. It fails with a clear message if there genuinely
        aren't enough 3200 students to meet it.
      </p>

      <UAlert
        v-if="suggestError"
        color="error"
        variant="subtle"
        title="Could not compute suggestions"
        :description="suggestError"
      />
      <UAlert
        v-if="usedFallbackThirdChoice"
        color="warning"
        variant="subtle"
        title="Some students required their 3rd choice"
        description="Capacity didn't allow 1st/2nd choice for everyone. Check which students show a 3rd-choice reason below before locking."
      />

      <UAlert
        v-if="lockError"
        color="error"
        variant="subtle"
        title="Lock/unlock failed"
        :description="lockError"
      />

      <p
        v-else-if="semesterId && !students3200ForDay.length"
        class="text-sm text-gray-500"
      >
        No 3200 students enrolled for this semester/day.
      </p>

      <ul v-else-if="semesterId" class="space-y-2">
        <li
          v-for="student in students3200ForDay"
          :key="student.id"
          class="flex flex-wrap items-center justify-between gap-2 rounded border border-gray-200 p-2 text-sm dark:border-gray-800"
        >
          <div>
            <div class="font-medium">
              {{ student.firstName }} {{ student.lastName }} ({{ student.netID }})
            </div>
            <div v-if="!lockedMembershipFor(student) && selectionLabel(student)" class="text-primary-500">
              {{ selectionLabel(student) }}
            </div>
            <div
              v-else-if="!lockedMembershipFor(student) && noChoiceStudentIds.has(student.id)"
              class="text-amber-500"
            >
              No bid submitted. Needs manual placement.
            </div>
          </div>

          <div v-if="lockedMembershipFor(student)" class="flex items-center gap-2">
            <UBadge color="primary" variant="subtle">
              Locked: {{ lockedMembershipFor(student)!.Team.Project.name }}
            </UBadge>
            <UButton
              label="Unlock"
              size="xs"
              color="neutral"
              variant="soft"
              :loading="lockingKey === lockedMembershipFor(student)!.id"
              @click="unlockMembership(lockedMembershipFor(student)!.id)"
            />
          </div>
          <div v-else class="flex items-center gap-2">
            <USelect
              v-model="lockSelections[student.id]"
              :items="teamOptionsFor(student)"
              placeholder="Choose team"
              class="w-64"
            />
            <UButton
              label="Lock"
              size="xs"
              :disabled="!lockSelections[student.id]"
              :loading="lockingKey === student.id"
              @click="lockStudent(student.id)"
            />
          </div>
        </li>
      </ul>
    </section>

    <section v-if="activeSection === 4" class="space-y-4">
      <h2 class="text-lg font-semibold">4. Team Generation</h2>


      <div class="grid grid-cols-2 gap-4">
        <UFormField label="Min Team Size">
          <UInputNumber v-model="config.min_team_size" :min="1" class="w-full" />
        </UFormField>
        <UFormField label="Max Team Size">
          <UInputNumber v-model="config.max_team_size" :min="config.min_team_size" class="w-full" />
        </UFormField>
      </div>

      <div class="grid grid-cols-2 gap-2">
        <UCheckbox
          v-model="config.prioritize_returning_students"
          label="Prioritize returning students"
        />
        <UCheckbox
          v-model="config.prioritize_3200_first_choice"
          label="Prioritize 3200 first choice"
        />
        <UCheckbox v-model="config.balance_gender" label="Balance gender" />
      </div>

      <p class="text-sm text-gray-500">
        3200-per-project range is set in step 3 (currently
        {{ minUpperclassPerProject }} to {{ maxUpperclassPerProject }}) and applies here too.
      </p>

      <UButton
        label="Generate Teams"
        icon="i-heroicons-sparkles"
        :loading="generating"
        :disabled="!canGenerate || generating"
        @click="generateTeams"
      />
      <p v-if="!canGenerate" class="text-sm text-gray-500">
        Select a semester/day with at least one team in step 1 first.
      </p>

      <UAlert
        v-if="generateError"
        color="error"
        variant="subtle"
        title="Team generation failed"
        :description="generateError"
      />

      <div v-if="generateResult" class="space-y-4">
        <UAlert color="success" variant="subtle" title="Teams generated successfully" />
        <p class="text-sm text-gray-500">
          Happy with a team? Lock it, then hit Generate Teams again. Locked teams are
          left exactly as is, and the algorithm only re-solves for everyone else. Keep
          locking and regenerating as many rounds as you want.
        </p>

        <UCheckbox
          v-model="showChoiceRank"
          label="Show each student's choice rank for their assigned project"
        />

        <UAlert
          v-if="teamLockError"
          color="error"
          variant="subtle"
          title="Could not update team lock"
          :description="teamLockError"
        />
        <UAlert
          v-if="moveError"
          color="error"
          variant="subtle"
          title="Could not move student"
          :description="moveError"
        />

        <div class="space-y-2">
          <div
            v-for="[teamId, meta] in Object.entries(generateResult.teamMeta)"
            :key="teamId"
            class="rounded border border-gray-200 p-3 dark:border-gray-800"
          >
            <div class="mb-1 flex items-center justify-between gap-2">
              <div class="flex items-center gap-2 font-medium">
                {{ meta.projectName }}
                <UBadge v-if="isTeamLocked(teamId)" color="primary" variant="subtle" size="sm">Locked</UBadge>
                <UBadge v-if="isPinned(teamId)" color="warning" variant="subtle" size="sm">Pinned</UBadge>
              </div>
              <div class="flex items-center gap-2">
                <UButton
                  :icon="isPinned(teamId) ? 'i-heroicons-bookmark-slash' : 'i-heroicons-bookmark'"
                  :label="isPinned(teamId) ? 'Unpin' : 'Pin'"
                  size="xs"
                  color="warning"
                  variant="ghost"
                  @click="togglePin(teamId)"
                />
                <UButton
                  :label="isTeamLocked(teamId) ? 'Unlock Team' : 'Lock Team'"
                  size="xs"
                  :color="isTeamLocked(teamId) ? 'neutral' : 'primary'"
                  :variant="isTeamLocked(teamId) ? 'soft' : 'solid'"
                  :loading="lockingTeamKey === teamId"
                  @click="toggleTeamLock(teamId, !isTeamLocked(teamId))"
                />
              </div>
            </div>
            <ul class="space-y-1 text-sm text-gray-600 dark:text-gray-400">
              <li v-for="student in studentsFor(teamId)" :key="student.netID">
                <div class="flex items-center justify-between gap-2">
                  <span>
                    {{ student.firstName }} {{ student.lastName }} ({{ student.netID }})
                    <UBadge v-if="is3200(student)" color="primary" variant="subtle" size="sm">3200</UBadge>
                    <span v-if="showChoiceRank" class="text-gray-400">
                      ({{ choiceRankLabel(student, teamId) }})
                    </span>
                  </span>
                  <UButton
                    label="Move"
                    size="xs"
                    color="neutral"
                    variant="ghost"
                    @click="openMovePanel = openMovePanel === student.id ? null : student.id"
                  />
                </div>
                <div
                  v-if="openMovePanel === student.id"
                  class="ml-2 mt-1 space-y-2 rounded border border-gray-200 p-2 text-xs dark:border-gray-800"
                >
                  <div class="text-gray-500">
                    Major: {{ majorFor(student) }}. Choices:
                    {{ resultChoiceLabels(student).join(', ') || 'none' }}
                  </div>
                  <div class="flex flex-wrap items-center gap-2">
                    <USelect
                      v-model="moveSelections[student.id]"
                      :items="resultTeamOptionsFor(student)"
                      placeholder="Move to..."
                      class="w-56"
                    />
                    <UCheckbox v-model="moveLockChecked[student.id]" label="Lock in new team" />
                    <UButton
                      label="Confirm Move"
                      size="xs"
                      :disabled="!moveSelections[student.id] || moveSelections[student.id] === teamId"
                      :loading="movingKey === student.id"
                      @click="moveStudent(student.id, teamId)"
                    />
                  </div>
                </div>
              </li>
            </ul>
          </div>
        </div>
      </div>
    </section>

    <section v-if="activeSection === 5" class="space-y-4">
      <h2 class="text-lg font-semibold">5. Team Board</h2>

        <p class="text-sm text-gray-500">
          Drag teams between columns. Set teams aside in the Workspace, then drag students
          between them. While you drag a student, every Workspace team shows how that student
          ranked it. Click a student to see their full ranking. Moving a student keeps their
          lock state. Dropping a team in Locked locks every member, so Generate Teams leaves it
          alone.
        </p>
        <p v-if="!semesterId" class="text-sm text-gray-500">Select a semester in step 1 first.</p>

        <UAlert
          v-if="boardError"
          color="error"
          variant="subtle"
          title="Could not update the board"
          :description="boardError"
        />

        <div v-if="semesterId" class="grid grid-cols-1 gap-4 lg:grid-cols-4">
          <div
            v-for="col in BOARD_ZONES"
            :key="col.zone"
            :class="[
              'min-h-48 space-y-3 rounded-lg border-2 border-dashed p-3 transition-colors',
              col.zone === 'workspace' ? 'lg:col-span-2' : '',
              boardDropTarget === col.zone
                ? 'border-primary-500 bg-primary-50 dark:bg-primary-950/40'
                : canDropTeamOn(col.zone)
                  ? 'border-primary-300 dark:border-primary-800'
                  : 'border-gray-200 dark:border-gray-800',
              boardBusy ? 'opacity-60' : '',
            ]"
            @dragover="onZoneDragOver($event, col.zone)"
            @dragleave="boardDropTarget === col.zone && (boardDropTarget = null)"
            @drop.prevent="onZoneDrop(col.zone)"
          >
            <div>
              <div class="flex items-center gap-2 font-medium">
                <UIcon v-if="col.zone === 'locked'" name="i-heroicons-lock-closed" />
                <UIcon v-else-if="col.zone === 'workspace'" name="i-heroicons-squares-2x2" />
                {{ col.title }}
                <span class="text-sm text-gray-400">({{ boardTeamsIn(col.zone).length }})</span>
              </div>
              <p class="text-xs text-gray-500">{{ col.hint }}</p>
            </div>

            <div :class="col.zone === 'workspace' ? 'grid gap-3 sm:grid-cols-2' : 'space-y-3'">
              <div
                v-for="team in boardTeamsIn(col.zone)"
                :key="team.id"
                draggable="true"
                :class="[
                  'cursor-grab space-y-2 rounded-lg border bg-white p-2 text-sm shadow-sm active:cursor-grabbing dark:bg-gray-900',
                  boardDropTarget === team.id
                    ? 'border-primary-500 ring-2 ring-primary-500'
                    : col.zone === 'locked'
                      ? 'border-primary-300 dark:border-primary-800'
                      : 'border-gray-200 dark:border-gray-800',
                  boardDrag?.kind === 'team' && boardDrag.teamId === team.id ? 'opacity-40' : '',
                ]"
                @dragstart="onBoardDragStart($event, { kind: 'team', teamId: team.id })"
                @dragend="onBoardDragEnd"
                @dragover="onTeamDragOver($event, team.id)"
                @dragleave="boardDropTarget === team.id && (boardDropTarget = null)"
                @drop.prevent.stop="onTeamDrop(team.id)"
              >
                <div class="flex items-start justify-between gap-2">
                  <div>
                    <div class="font-medium">{{ team.projectName }}</div>
                    <div class="text-xs text-gray-500">
                      <span :class="team.memberCount > config.max_team_size ? 'text-red-500' : ''">
                        {{ team.memberCount }}/{{ config.max_team_size }} members
                      </span>
                      ·
                      <span
                        :class="
                          boardCount3200(team.id) < minUpperclassPerProject ? 'text-red-500' : ''
                        "
                      >
                        {{ boardCount3200(team.id) }} × 3200
                      </span>
                    </div>
                  </div>
                  <UBadge
                    v-if="boardDrag?.kind === 'student' && canDropStudentOn(team.id)"
                    :color="rankColor(boardRank(boardDrag.studentId, team.projectId))"
                    variant="solid"
                    size="sm"
                  >
                    {{ rankText(boardRank(boardDrag.studentId, team.projectId)) }}
                  </UBadge>
                </div>

                <ul v-if="col.zone !== 'pool'" class="space-y-1">
                  <li
                    v-for="m in membershipsForTeam(team.id)"
                    :key="m.id"
                    :draggable="col.zone === 'workspace'"
                    :class="[
                      'rounded border border-gray-100 px-2 py-1 dark:border-gray-800',
                      col.zone === 'workspace' ? 'cursor-grab hover:border-primary-300' : '',
                      boardDrag?.kind === 'student' && boardDrag.studentId === m.studentId
                        ? 'opacity-40'
                        : '',
                    ]"
                    @dragstart.stop="
                      onBoardDragStart($event, {
                        kind: 'student',
                        studentId: m.studentId,
                        fromTeamId: team.id,
                      })
                    "
                    @dragend.stop="onBoardDragEnd"
                    @click.stop="
                      expandedBoardStudent = expandedBoardStudent === m.id ? null : m.id
                    "
                  >
                    <div class="flex items-center justify-between gap-2">
                      <span class="flex items-center gap-1">
                        <UIcon
                          v-if="m.locked && col.zone !== 'locked'"
                          name="i-heroicons-lock-closed"
                          class="text-primary-500"
                        />
                        {{ m.Student.firstName }} {{ m.Student.lastName }}
                        <UBadge v-if="boardIs3200(m.studentId)" color="primary" variant="subtle" size="sm">
                          3200
                        </UBadge>
                      </span>
                      <UBadge
                        :color="rankColor(boardRank(m.studentId, team.projectId))"
                        variant="subtle"
                        size="sm"
                      >
                        {{ rankText(boardRank(m.studentId, team.projectId)) }}
                      </UBadge>
                    </div>
                    <ol
                      v-if="expandedBoardStudent === m.id"
                      class="mt-1 space-y-0.5 border-t border-gray-100 pt-1 text-xs dark:border-gray-800"
                    >
                      <li v-if="!boardChoiceList(m.studentId).length" class="text-amber-500">
                        No bid submitted.
                      </li>
                      <li
                        v-for="choice in boardChoiceList(m.studentId)"
                        :key="choice.rank"
                        :class="
                          choice.name === team.projectName
                            ? 'font-medium text-primary-500'
                            : choice.inWorkspace
                              ? 'text-gray-700 dark:text-gray-300'
                              : 'text-gray-400'
                        "
                      >
                        {{ choice.rank }}. {{ choice.name }}
                        <span v-if="choice.name === team.projectName">(current)</span>
                        <span v-else-if="choice.inWorkspace">(in Workspace)</span>
                      </li>
                    </ol>
                  </li>
                  <li v-if="!team.memberCount" class="text-xs text-gray-400">No members yet.</li>
                </ul>
              </div>

              <p v-if="!boardTeamsIn(col.zone).length" class="text-xs text-gray-400">
                {{ col.zone === 'pool' ? 'No other teams.' : 'Drop teams here.' }}
              </p>
            </div>
          </div>
        </div>
    </section>

    <div class="flex justify-between gap-2 border-t border-gray-200 pt-4 dark:border-gray-800">
      <UButton
        v-if="activeSection > 1"
        label="Back"
        icon="i-heroicons-arrow-left"
        color="neutral"
        variant="soft"
        @click="activeSection--"
      />
      <span v-else />
      <UButton
        v-if="activeSection < STEP_COUNT"
        label="Next"
        trailing-icon="i-heroicons-arrow-right"
        :disabled="!semesterId"
        @click="activeSection++"
      />
    </div>
  </UContainer>
</template>
