<script lang="ts">
	import { goto } from '$app/navigation'
	import type { Job } from '$lib/gen'
	import { displayDate, msToSec, truncateHash, truncateRev } from '$lib/utils'
	import { Badge, Button } from '../common'
	import ScheduleEditor from '../ScheduleEditor.svelte'
	import BarsStaggered from '$lib/components/icons/BarsStaggered.svelte'

	import {
		Calendar,
		Check,
		FastForward,
		Folder,
		Hourglass,
		ListFilter,
		Play,
		X
	} from 'lucide-svelte'
	import { createEventDispatcher } from 'svelte'
	import TimeAgo from '../TimeAgo.svelte'
	import { forLater } from '$lib/forLater'
	import { twMerge } from 'tailwind-merge'
	import Portal from 'svelte-portal'

	const dispatch = createEventDispatcher()

	export let job: Job
	export let selectedId: string | undefined = undefined
	export let containerWidth: number = 0
	export let containsLabel: boolean = false
	export let activeLabel: string | null

	let scheduleEditor: ScheduleEditor
</script>

<Portal>
	<ScheduleEditor on:update={() => goto('/schedules')} bind:this={scheduleEditor} />
</Portal>

<!-- svelte-ignore a11y-click-events-have-key-events -->
<!-- svelte-ignore a11y-no-static-element-interactions -->
<div
	class={twMerge(
		'hover:bg-surface-hover cursor-pointer',
		selectedId === job.id ? 'bg-blue-50 dark:bg-blue-900/50' : '',
		'flex flex-row items-center h-full'
	)}
	style="width: {containerWidth}px"
	on:click={() => {
		dispatch('select')
	}}
>
	<div class="w-1/12 flex justify-center">
		{#if 'success' in job && job.success}
			{#if job.is_skipped}
				<Badge color="green" rounded>
					<FastForward size={14} />
				</Badge>
			{:else}
				<Badge color="green" baseClass="!px-1.5">
					<Check size={14} />
				</Badge>
			{/if}
		{:else if 'success' in job}
			<Badge color="red" baseClass="!px-1.5">
				<X size={14} />
			</Badge>
		{:else if 'running' in job && job.running}
			<Badge color="yellow" baseClass="!px-1.5">
				<Play size={14} />
			</Badge>
		{:else if job && 'running' in job && job.scheduled_for && forLater(job.scheduled_for)}
			<Badge color="blue" baseClass="!px-1.5">
				<Calendar size={14} />
			</Badge>
		{:else}
			<Badge baseClass="!px-1.5">
				<Hourglass size={14} />
			</Badge>
		{/if}
	</div>

	<div class="w-4/12 flex justify-start">
		<div class="flex flex-row items-center gap-1 text-gray-500 dark:text-gray-300 text-2xs">
			{#if job}
				{#if 'started_at' in job && job.started_at}
					Started <TimeAgo date={job.started_at ?? ''} />
					{#if job && 'duration_ms' in job && job.duration_ms != undefined}
						(Ran in {msToSec(
							job.duration_ms
						)}s{#if job.job_kind == 'flow' || job.job_kind == 'flowpreview'}&nbsp;total{/if})
					{/if}
				{:else if `scheduled_for` in job && job.scheduled_for && forLater(job.scheduled_for)}
					Scheduled for {displayDate(job.scheduled_for)}
				{:else}
					Waiting for executor (created <TimeAgo date={job.created_at || ''} />)
				{/if}
			{/if}
		</div>
	</div>

	<div class="w-4/12 flex justify-start flex-col">
		<div class="flex flex-row text-sm">
			{#if job === undefined}
				No job found
			{:else}
				<div class="flex flex-row gap-1 min-w-0">
					<div class="whitespace-nowrap text-xs font-semibold truncate">
						{#if job.script_path}
							<div class="flex flex-row gap-1 items-center">
								<a
									href="/run/{job.id}?workspace={job.workspace_id}"
									class="truncate w-30 dark:text-blue-400"
								>
									{job.script_path}
								</a>
								<Button
									size="xs2"
									color="light"
									on:click={() => {
										dispatch('filterByPath', job.script_path)
									}}
								>
									<ListFilter size={10} />
								</Button>
								{#if job.script_path?.startsWith('f/')}
									<Button
										size="xs2"
										color="light"
										on:click={() => {
											// split script_path by / and get the second element
											const folder = job.script_path?.split('/')[1]

											dispatch('filterByFolder', folder)
										}}
									>
										<Folder size={10} />
									</Button>
								{/if}
							</div>
						{:else if 'job_kind' in job && job.job_kind == 'preview'}
							<a href="/run/{job.id}?workspace={job.workspace_id}">Preview without path </a>
						{:else if 'job_kind' in job && job.job_kind == 'dependencies'}
							<a href="/run/{job.id}?workspace={job.workspace_id}">
								lock deps of {truncateHash(job.script_hash ?? '')}
							</a>
						{:else if 'job_kind' in job && job.job_kind == 'identity'}
							<a href="/run/{job.id}?workspace={job.workspace_id}">no op</a>
						{/if}
					</div>
				</div>
			{/if}
		</div>

		{#if job && job.parent_job}
			{#if job.is_flow_step}
				<div class="flex flex-row gap-1 items-center">
					<BarsStaggered class="text-secondary" size={14} />
					<span class="mx-1 text-xs">
						Step of flow <a href={`/run/${job.parent_job}?workspace=${job.workspace_id}`}>
							{truncateRev(job.parent_job, 6)}
						</a>
					</span>
				</div>
			{:else}
				<div class="flex flex-row gap-1 items-center">
					<span class="text-2xs text-tertiary truncate">
						parent <a href={`/run/${job.parent_job}?workspace=${job.workspace_id}`}>
							{truncateRev(job.parent_job, 10)}
						</a>
					</span>
				</div>
			{/if}
		{/if}
	</div>
	{#if containsLabel}
		<div class="w-3/12 flex justify-start">
			{#if job && job?.['labels']}
				<div class="flex flex-row items-center gap-1 overflow-x-auto">
					{#if Array.isArray(job?.['labels'])}
						{#each job?.['labels'] as label}
							<Button
								variant="border"
								size="xs2"
								btnClasses={twMerge(
									activeLabel == label ? 'bg-blue-50 dark:bg-blue-900/50' : '',
									'!text-2xs !font-normal truncate max-w-28'
								)}
								color="light"
								on:click={() => {
									dispatch('filterByLabel', label)
								}}
							>
								{label}
								<ListFilter size={10} />
							</Button>
						{/each}
					{/if}
				</div>
			{/if}
		</div>
	{/if}
	<div class="w-3/12 flex justify-start">
		{#if job && job.schedule_path}
			<div class="flex flex-row items-center gap-1">
				<Calendar size={14} />
				<Button
					size="xs2"
					color="light"
					btnClasses="font-normal"
					on:click={() => scheduleEditor?.openEdit(job.schedule_path ?? '', job.job_kind == 'flow')}
				>
					{job.schedule_path}
				</Button>
			</div>
		{:else}
			<div class="flex flex-row gap-1 items-center">
				<div class="text-xs">
					{job.created_by}
				</div>
				<Button
					size="xs2"
					color="light"
					on:click={() => {
						dispatch('filterByUser', job.created_by)
					}}
				>
					<ListFilter size={10} />
				</Button>
			</div>
		{/if}
	</div>
</div>
