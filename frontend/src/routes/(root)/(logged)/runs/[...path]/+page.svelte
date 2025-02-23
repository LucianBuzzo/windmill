<script lang="ts">
	import {
		JobService,
		type Job,
		type CompletedJob,
		UserService,
		FolderService,
		ScriptService,
		FlowService
	} from '$lib/gen'

	import { page } from '$app/stores'
	import { sendUserToast } from '$lib/toast'
	import { superadmin, userStore, workspaceStore } from '$lib/stores'
	import { Button, Drawer, DrawerContent, Skeleton } from '$lib/components/common'
	import RunChart from '$lib/components/RunChart.svelte'

	import JobPreview from '$lib/components/runs/JobPreview.svelte'
	import Tooltip from '$lib/components/Tooltip.svelte'
	import CalendarPicker from '$lib/components/common/calendarPicker/CalendarPicker.svelte'

	import RunsTable from '$lib/components/runs/RunsTable.svelte'
	import SplitPanesWrapper from '$lib/components/splitPanes/SplitPanesWrapper.svelte'
	import { Pane, Splitpanes } from 'svelte-splitpanes'
	import RunsFilter from '$lib/components/runs/RunsFilter.svelte'
	import Toggle from '$lib/components/Toggle.svelte'
	import ConfirmationModal from '$lib/components/common/confirmationModal/ConfirmationModal.svelte'
	import type { Tweened } from 'svelte/motion'
	import RunsQueue from '$lib/components/runs/RunsQueue.svelte'
	import { twMerge } from 'tailwind-merge'
	import ManuelDatePicker from '$lib/components/runs/ManuelDatePicker.svelte'
	import JobLoader from '$lib/components/runs/JobLoader.svelte'
	import { Calendar, Clock } from 'lucide-svelte'

	let jobs: Job[] | undefined
	let selectedId: string | undefined = undefined
	let selectedWorkspace: string | undefined = undefined

	// All Filters
	// Filter by
	let path: string | null = $page.params.path
	let user: string | null = $page.url.searchParams.get('user')
	let folder: string | null = $page.url.searchParams.get('folder')
	let label: string | null = $page.url.searchParams.get('label')
	// Rest of filters handled by RunsFilter
	let success: 'running' | 'success' | 'failure' | undefined = ($page.url.searchParams.get(
		'success'
	) ?? undefined) as 'running' | 'success' | 'failure' | undefined
	let isSkipped: boolean | undefined =
		$page.url.searchParams.get('is_skipped') != undefined
			? $page.url.searchParams.get('is_skipped') == 'true'
			: false

	let showSchedules: boolean =
		$page.url.searchParams.get('show_schedules') != undefined
			? $page.url.searchParams.get('show_schedules') == 'true'
			: localStorage.getItem('show_schedules_in_run') == 'false'
			? false
			: true
	let showFutureJobs: boolean =
		$page.url.searchParams.get('show_future_jobs') != undefined
			? $page.url.searchParams.get('show_future_jobs') == 'true'
			: localStorage.getItem('show_future_jobs') == 'false'
			? false
			: true

	let argFilter: any = $page.url.searchParams.get('arg')
		? JSON.parse(decodeURIComponent($page.url.searchParams.get('arg') ?? '{}'))
		: undefined
	let resultFilter: any = $page.url.searchParams.get('result')
		? JSON.parse(decodeURIComponent($page.url.searchParams.get('result') ?? '{}'))
		: undefined

	// Handled on the main page
	let minTs = $page.url.searchParams.get('min_ts') ?? undefined
	let maxTs = $page.url.searchParams.get('max_ts') ?? undefined
	let schedulePath = $page.url.searchParams.get('schedule_path') ?? undefined
	let jobKindsCat = $page.url.searchParams.get('job_kinds') ?? 'runs'
	let allWorkspaces = $page.url.searchParams.get('all_workspaces') == 'true' ?? false

	let queue_count: Tweened<number> | undefined = undefined
	let jobKinds: string | undefined = undefined
	let loading: boolean = false
	let paths: string[] = []
	let usernames: string[] = []
	let folders: string[] = []
	let completedJobs: CompletedJob[] | undefined = undefined
	let argError = ''
	let resultError = ''
	let filterTimeout: NodeJS.Timeout | undefined = undefined
	let selectedManualDate = 0
	let autoRefresh: boolean = true
	let runDrawer: Drawer
	let cancelAllJobs = false
	let innerWidth = window.innerWidth
	let jobLoader: JobLoader | undefined = undefined

	let manualDatePicker: ManuelDatePicker

	$: (user ||
		label ||
		folder ||
		path ||
		success !== undefined ||
		isSkipped ||
		showSchedules ||
		showFutureJobs ||
		argFilter ||
		resultFilter ||
		schedulePath ||
		jobKindsCat ||
		minTs ||
		maxTs ||
		allWorkspaces ||
		$workspaceStore) &&
		setQuery()

	function setQuery() {
		let searchParams = new URLSearchParams()

		if (user) {
			searchParams.set('user', user)
		} else {
			searchParams.delete('user')
		}

		if (folder) {
			searchParams.set('folder', folder)
		} else {
			searchParams.delete('folder')
		}

		if (success !== undefined) {
			searchParams.set('success', success.toString())
		} else {
			searchParams.delete('success')
		}

		if (isSkipped) {
			searchParams.set('is_skipped', isSkipped.toString())
		} else {
			searchParams.delete('is_skipped')
		}

		if (showSchedules) {
			searchParams.set('show_schedules', showSchedules.toString())
		} else {
			searchParams.delete('show_schedules')
		}

		if (showFutureJobs) {
			searchParams.set('show_future_jobs', showFutureJobs.toString())
		} else {
			searchParams.delete('show_future_jobs')
		}

		if (allWorkspaces && $workspaceStore == 'admins') {
			searchParams.set('all_workspaces', allWorkspaces.toString())
			searchParams.set('workspace', 'admins')
		} else {
			searchParams.delete('all_workspaces')
		}

		// ArgFilter is an object. Encode it to a string
		if (argFilter) {
			searchParams.set('arg', encodeURIComponent(JSON.stringify(argFilter)))
		} else {
			searchParams.delete('arg')
		}

		if (resultFilter) {
			searchParams.set('result', encodeURIComponent(JSON.stringify(resultFilter)))
		} else {
			searchParams.delete('result')
		}
		if (schedulePath) {
			searchParams.set('schedule_path', schedulePath)
		} else {
			searchParams.delete('schedule_path')
		}
		if (jobKindsCat != 'runs') {
			searchParams.set('job_kinds', jobKindsCat)
		} else {
			searchParams.delete('job_kinds')
		}

		if (minTs) {
			searchParams.set('min_ts', minTs)
		} else {
			searchParams.delete('min_ts')
		}

		if (maxTs) {
			searchParams.set('max_ts', maxTs)
		} else {
			searchParams.delete('max_ts')
		}

		if (label) {
			searchParams.set('label', label)
		} else {
			searchParams.delete('label')
		}

		let newPath = path ? `/${path}` : '/'
		let newUrl = `/runs${newPath}?${searchParams.toString()}`
		history.replaceState(history.state, '', newUrl.toString())
	}

	function reloadJobsWithoutFilterError() {
		if (resultError == '' && argError == '') {
			filterTimeout && clearTimeout(filterTimeout)
			filterTimeout = setTimeout(() => {
				jobLoader?.loadJobs(minTs, maxTs, true)
			}, 2000)
		}
	}

	function reset() {
		minTs = undefined
		maxTs = undefined

		autoRefresh = true
		jobs = undefined
		completedJobs = undefined
		selectedManualDate = 0
		selectedId = undefined
		selectedWorkspace = undefined
		jobLoader?.loadJobs(minTs, maxTs, true)
	}

	async function loadUsernames(): Promise<void> {
		usernames = await UserService.listUsernames({ workspace: $workspaceStore! })
	}

	async function loadFolders(): Promise<void> {
		folders = await FolderService.listFolders({
			workspace: $workspaceStore!
		}).then((x) => x.map((y) => y.name))
	}

	async function loadPaths() {
		const npaths_scripts = await ScriptService.listScriptPaths({ workspace: $workspaceStore ?? '' })
		const npaths_flows = await FlowService.listFlowPaths({ workspace: $workspaceStore ?? '' })
		paths = npaths_scripts.concat(npaths_flows).sort()
	}

	$: if ($workspaceStore) {
		loadUsernames()
		loadFolders()
		loadPaths()
	}

	function filterByPath(e: CustomEvent<string>) {
		path = e.detail
		user = null
		folder = null
		label = null
	}

	function filterByUser(e: CustomEvent<string>) {
		path = null
		folder = null
		user = e.detail
		label = null
	}

	function filterByFolder(e: CustomEvent<string>) {
		path = null
		user = null
		folder = e.detail
		label = null
	}

	function filterByLabel(e: CustomEvent<string>) {
		path = null
		user = null
		folder = null
		label = e.detail
	}

	let calendarChangeTimeout: NodeJS.Timeout | undefined = undefined
</script>

<JobLoader
	{allWorkspaces}
	bind:jobs
	{user}
	{folder}
	{path}
	{label}
	{success}
	{isSkipped}
	{argFilter}
	{resultFilter}
	{showSchedules}
	{showFutureJobs}
	{schedulePath}
	{jobKindsCat}
	computeMinAndMax={manualDatePicker?.computeMinMax}
	bind:minTs
	bind:maxTs
	{jobKinds}
	bind:queue_count
	{autoRefresh}
	bind:completedJobs
	{argError}
	{resultError}
	bind:loading
	bind:this={jobLoader}
/>

<ConfirmationModal
	title="Confirm cancelling all jobs"
	confirmationText="Cancel all jobs"
	open={cancelAllJobs}
	on:confirmed={async () => {
		cancelAllJobs = false
		let uuids = await JobService.cancelAll({ workspace: $workspaceStore ?? '' })
		jobLoader?.loadJobs(minTs, maxTs, true, true)
		sendUserToast(`Canceled ${uuids.length} jobs`)
	}}
	on:canceled={() => {
		cancelAllJobs = false
	}}
/>

<Drawer bind:this={runDrawer}>
	<DrawerContent title="Run details" on:close={runDrawer.closeDrawer}>
		{#if selectedId}
			<JobPreview blankLink id={selectedId} workspace={selectedWorkspace} />
		{/if}
	</DrawerContent>
</Drawer>

<svelte:window bind:innerWidth />

{#if innerWidth > 1280}
	<div class="w-full h-screen">
		<div class="px-2">
			<div class="flex items-center space-x-2 flex-row justify-between">
				<div class="flex flex-row flex-wrap justify-between py-2 my-4 px-4 gap-1 items-center">
					<h1
						class={twMerge(
							'!text-2xl font-semibold leading-6 tracking-tight',
							$userStore?.operator ? 'pl-10' : ''
						)}
					>
						Runs
					</h1>

					<Tooltip
						documentationLink="https://www.windmill.dev/docs/core_concepts/monitor_past_and_future_runs"
					>
						All past and schedule executions of scripts and flows, including previews. You only see
						your own runs or runs of groups you belong to unless you are an admin.
					</Tooltip>
				</div>
				<RunsFilter
					bind:isSkipped
					bind:user
					bind:folder
					bind:label
					bind:path
					bind:success
					bind:argFilter
					bind:resultFilter
					bind:argError
					bind:resultError
					bind:jobKindsCat
					bind:allWorkspaces
					on:change={reloadJobsWithoutFilterError}
					{usernames}
					{folders}
					{paths}
				/>
			</div>
		</div>

		<div class="p-2 w-full">
			<RunChart
				minTimeSet={minTs}
				maxTimeSet={maxTs}
				maxIsNow={maxTs == undefined}
				jobs={completedJobs}
				on:zoom={async (e) => {
					minTs = e.detail.min.toISOString()
					maxTs = e.detail.max.toISOString()
					jobLoader?.loadJobs(minTs, maxTs, true)
				}}
			/>
		</div>
		<div class="flex flex-col gap-1 md:flex-row w-full p-4">
			<div class="flex gap-2 grow flex-row">
				<RunsQueue {queue_count} {allWorkspaces} />
				<Button
					size="xs"
					color="light"
					variant="contained"
					title="Require to be an admin. Cancel all jobs in queue"
					disabled={!$userStore?.is_admin && !$superadmin}
					on:click={async () => (cancelAllJobs = true)}>Cancel All</Button
				>
			</div>
			<div class="relative flex gap-2 items-center pr-8 w-40">
				<Toggle
					size="xs"
					bind:checked={showSchedules}
					on:change={() => {
						localStorage.setItem('show_schedules_in_run', showSchedules ? 'true' : 'false')
					}}
				/>
				<span class="text-xs absolute -top-4">CRON Schedules</span>

				<Calendar size={16} />
			</div>
			<div class="relative flex gap-2 items-center pr-8 w-40">
				<span class="text-xs absolute -top-4">Planned later</span>
				<Toggle
					size="xs"
					bind:checked={showFutureJobs}
					on:change={() => {
						localStorage.setItem('show_future_jobs', showFutureJobs ? 'true' : 'false')
					}}
				/>
				<Clock size={16} />
			</div>
			<div class="flex flex-row gap-1 w-full max-w-lg">
				<div class="relative w-full">
					<div class="flex gap-1 relative w-full">
						<span class="text-xs absolute -top-4">Min datetime</span>

						<input
							type="text"
							value={minTs
								? new Date(minTs).toLocaleString()
								: 'zoom x axis to set min (drag with ctrl)'}
							disabled
						/>

						<CalendarPicker
							date={minTs}
							label="Min datetimes"
							on:change={async ({ detail }) => {
								minTs = new Date(detail).toISOString()
								calendarChangeTimeout && clearTimeout(calendarChangeTimeout)
								calendarChangeTimeout = setTimeout(() => {
									jobLoader?.loadJobs(minTs, maxTs, true)
								}, 1000)
							}}
						/>
					</div>
				</div>
				<div class="relative w-full">
					<div class="flex gap-1 relative w-full">
						<span class="text-xs absolute -top-4">Max datetime</span>
						<input
							type="text"
							value={maxTs ? new Date(maxTs).toLocaleString() : 'zoom x axis to set max'}
							disabled
						/>
						<CalendarPicker
							date={maxTs}
							label="Max datetimes"
							on:change={async ({ detail }) => {
								maxTs = new Date(detail).toISOString()
								calendarChangeTimeout && clearTimeout(calendarChangeTimeout)
								calendarChangeTimeout = setTimeout(() => {
									jobLoader?.loadJobs(minTs, maxTs, true)
								}, 1000)
							}}
						/>
					</div>
				</div>
			</div>
			<div class="flex flex-row gap-2 items-center">
				<Button size="xs" color="light" variant="border" on:click={reset}>Reset</Button>
				<ManuelDatePicker
					on:loadJobs={() => {
						jobLoader?.loadJobs(minTs, maxTs, true, true)
					}}
					bind:minTs
					bind:maxTs
					bind:selectedManualDate
					{loading}
					bind:this={manualDatePicker}
				/>
				<Toggle
					size="xs"
					bind:checked={autoRefresh}
					options={{ right: 'Auto-refresh' }}
					textClass="whitespace-nowrap"
				/>
			</div>
		</div>

		<SplitPanesWrapper>
			<Splitpanes>
				<Pane size={60} minSize={40}>
					{#if jobs}
						<RunsTable
							{jobs}
							activeLabel={label}
							bind:selectedId
							bind:selectedWorkspace
							on:filterByPath={filterByPath}
							on:filterByUser={filterByUser}
							on:filterByFolder={filterByFolder}
							on:filterByLabel={filterByLabel}
						/>
					{:else}
						<div class="gap-1 flex flex-col">
							{#each new Array(8) as _}
								<Skeleton layout={[[3]]} />
							{/each}
						</div>
					{/if}
				</Pane>
				<Pane size={40} minSize={15} class="border-t">
					{#if selectedId}
						<JobPreview id={selectedId} workspace={selectedWorkspace} />
					{:else}
						<div class="text-xs m-4">No job selected</div>
					{/if}
				</Pane>
			</Splitpanes>
		</SplitPanesWrapper>
	</div>
{:else}
	<div class="flex flex-col h-screen">
		<div class="px-2">
			<div class="flex items-center space-x-2 flex-row justify-between">
				<div class="flex flex-row flex-wrap justify-between py-2 my-4 px-4 gap-1">
					<h1 class="!text-2xl font-semibold leading-6 tracking-tight"> Runs </h1>

					<Tooltip
						light
						documentationLink="https://www.windmill.dev/docs/core_concepts/monitor_past_and_future_runs"
						scale={0.9}
						wrapperClass="flex items-center"
					>
						All past and schedule executions of scripts and flows, including previews. You only see
						your own runs or runs of groups you belong to unless you are an admin.
					</Tooltip>
				</div>
				<RunsFilter
					bind:isSkipped
					{paths}
					{usernames}
					{folders}
					bind:jobKindsCat
					bind:folder
					bind:path
					bind:user
					bind:success
					bind:argFilter
					bind:resultFilter
					bind:argError
					bind:resultError
					bind:allWorkspaces
					mobile={true}
					on:change={reloadJobsWithoutFilterError}
				/>
			</div>
		</div>
		<div class="p-2 w-full">
			<RunChart
				minTimeSet={minTs}
				maxTimeSet={maxTs}
				maxIsNow={maxTs == undefined}
				jobs={completedJobs}
				on:zoom={async (e) => {
					minTs = e.detail.min.toISOString()
					maxTs = e.detail.max.toISOString()
					jobLoader?.loadJobs(minTs, maxTs, true)
				}}
			/>
		</div>
		<div class="flex flex-col gap-4 md:flex-row w-full p-4">
			<div class="flex items-center flex-row gap-2 grow">
				{#if queue_count}
					<RunsQueue {queue_count} {allWorkspaces} />
				{/if}
				<Button
					size="xs"
					color="light"
					variant="contained"
					title="Require to be an admin. Cancel all jobs in queue"
					disabled={!$userStore?.is_admin && !$superadmin}
					on:click={async () => (cancelAllJobs = true)}>Cancel All</Button
				>
			</div>
			<div class="flex gap-2 py-1">
				<div class="relative flex gap-2 items-center pr-8 w-40">
					<Toggle
						size="xs"
						bind:checked={showSchedules}
						on:change={() => {
							localStorage.setItem('show_schedules_in_run', showSchedules ? 'true' : 'false')
						}}
					/>
					<span class="text-xs absolute -top-4">Schedules</span>

					<Calendar size={16} />
				</div>
				<div class="relative flex gap-2 items-center pr-8 w-40">
					<span class="text-xs absolute -top-4">Planned later</span>
					<Toggle
						size="xs"
						bind:checked={showFutureJobs}
						on:change={() => {
							localStorage.setItem('show_future_jobs', showFutureJobs ? 'true' : 'false')
						}}
					/>
					<Clock size={16} />
				</div>
			</div>
			<div class="flex flex-row gap-1 w-full max-w-lg items-center">
				<div class="relative w-full">
					<div class="flex gap-1 relative w-full">
						<span class="text-xs absolute -top-4">Min datetime</span>

						<input
							type="text"
							value={minTs
								? new Date(minTs).toLocaleString()
								: 'zoom x axis to set min (drag with ctrl)'}
							disabled
						/>

						<CalendarPicker
							date={minTs}
							label="Min datetimes"
							on:change={async ({ detail }) => {
								minTs = new Date(detail).toISOString()
								calendarChangeTimeout && clearTimeout(calendarChangeTimeout)
								calendarChangeTimeout = setTimeout(() => {
									jobLoader?.loadJobs(minTs, maxTs, true)
								}, 1000)
							}}
						/>
					</div>
				</div>
				<div class="relative w-full">
					<div class="flex gap-1 relative w-full">
						<span class="text-xs absolute -top-4">Max datetime</span>
						<input
							type="text"
							value={maxTs ? new Date(maxTs).toLocaleString() : 'zoom x axis to set max'}
							disabled
						/>
						<CalendarPicker
							date={maxTs}
							label="Max datetimes"
							on:change={async ({ detail }) => {
								maxTs = new Date(detail).toISOString()
								calendarChangeTimeout && clearTimeout(calendarChangeTimeout)
								calendarChangeTimeout = setTimeout(() => {
									jobLoader?.loadJobs(minTs, maxTs, true)
								}, 1000)
							}}
						/>
					</div>
				</div>
			</div>
			<div class="flex flex-row gap-2 items-center">
				<Button size="xs" color="light" variant="border" on:click={reset}>Reset</Button>
				<ManuelDatePicker
					on:loadJobs={() => {
						jobLoader?.loadJobs(minTs, maxTs, true, true)
					}}
					bind:this={manualDatePicker}
					bind:minTs
					bind:maxTs
					bind:selectedManualDate
					{loading}
				/>

				<Toggle
					size="xs"
					bind:checked={autoRefresh}
					options={{ right: 'Auto-refresh' }}
					textClass="whitespace-nowrap"
				/>
			</div>
		</div>
		<div class="grow">
			<RunsTable
				activeLabel={label}
				{jobs}
				bind:selectedId
				bind:selectedWorkspace
				on:select={() => {
					runDrawer.openDrawer()
				}}
				on:filterByPath={filterByPath}
				on:filterByUser={filterByUser}
				on:filterByFolder={filterByFolder}
				on:filterByLabel={filterByLabel}
			/>
		</div>
	</div>
{/if}
