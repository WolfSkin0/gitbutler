<script lang="ts">
	import CardOverlay from '$components/CardOverlay.svelte';
	import ScrollableContainer from '$components/ConfigurableScrollableContainer.svelte';
	import Dropzone from '$components/Dropzone.svelte';
	import FileList from '$components/FileList.svelte';
	import FileListMode from '$components/FileListMode.svelte';
	import UnassignedFoldButton from '$components/UnassignedFoldButton.svelte';
	import WorktreeChangesSelectAll from '$components/WorktreeChangesSelectAll.svelte';
	import { createCommitStore } from '$lib/commits/contexts';
	import { UncommitDzHandler } from '$lib/commits/dropHandler';
	import { DiffService } from '$lib/hunks/diffService.svelte';
	import { AssignmentDropHandler } from '$lib/hunks/dropHandler';
	import { IdSelection } from '$lib/selection/idSelection.svelte';
	import { UncommittedService } from '$lib/selection/uncommittedService.svelte';
	import { StackService } from '$lib/stacks/stackService.svelte';
	import { UiState } from '$lib/state/uiState.svelte';
	import { TestId } from '$lib/testing/testIds';
	import { inject } from '@gitbutler/shared/context';
	import Badge from '@gitbutler/ui/Badge.svelte';

	import { isDefined } from '@gitbutler/ui/utils/typeguards';
	import { type Snippet } from 'svelte';
	import type { DropzoneHandler } from '$lib/dragging/handler';

	type Props = {
		projectId: string;
		stackId?: string;
		active: boolean;
		title: string;
		mode?: 'unassigned' | 'assigned';
		dropzoneVisible?: boolean;
		overflow?: boolean;
		onDropzoneActivated?: (activated: boolean) => void;
		emptyPlaceholder?: Snippet;
		onselect?: () => void;
		onscrollexists?: (exists: boolean) => void;
	};

	let {
		projectId,
		active,
		stackId,
		title,
		mode = 'unassigned',
		dropzoneVisible,
		overflow,
		onDropzoneActivated,
		emptyPlaceholder,
		onselect,
		onscrollexists
	}: Props = $props();

	const [uiState, stackService, diffService, uncommittedService, idSelection] = inject(
		UiState,
		StackService,
		DiffService,
		UncommittedService,
		IdSelection
	);

	const uncommitDzHandler = $derived(
		new UncommitDzHandler(projectId, stackService, uiState, stackId)
	);
	const unassignedSidebaFolded = $derived(uiState.global.unassignedSidebaFolded);

	const projectState = $derived(uiState.project(projectId));
	const exclusiveAction = $derived(projectState.exclusiveAction.current);
	const isCommitting = $derived(
		exclusiveAction?.type === 'commit' &&
			(exclusiveAction.stackId === stackId || stackId === undefined)
	);

	const changes = $derived(uncommittedService.changesByStackId(stackId || null));

	// TODO: Remove this after V3 transition complete.
	createCommitStore(undefined);

	let listMode: 'list' | 'tree' = $state('list');

	let scrollTopIsVisible = $state(true);

	const assignmentDZHandler = $derived(
		new AssignmentDropHandler(
			projectId,
			diffService,
			uncommittedService,
			stackId || null,
			idSelection
		)
	);

	function getDropzoneLabel(handler: DropzoneHandler | undefined): string {
		if (handler instanceof UncommitDzHandler) {
			return 'Uncommit changes';
		} else if (mode === 'assigned') {
			return 'Assign changes';
		} else {
			return 'Unassign changes';
		}
	}

	function foldUnnassignedView() {
		unassignedSidebaFolded.set(true);
	}
</script>

{#snippet fileList()}
	<div data-testid={TestId.UncommittedChanges_FileList} class="uncommitted-changes">
		<FileList
			draggableFiles
			allowUnselect
			selectionId={{ type: 'worktree', stackId }}
			showCheckboxes={isCommitting}
			changes={changes.current}
			{projectId}
			{listMode}
			{active}
			{stackId}
			{onselect}
		/>
	</div>
{/snippet}

<Dropzone
	handlers={[uncommitDzHandler, assignmentDZHandler].filter(isDefined)}
	maxHeight
	onActivated={onDropzoneActivated}
	{overflow}
>
	{#snippet overlay({ hovered, activated, handler })}
		<CardOverlay
			visible={dropzoneVisible}
			{hovered}
			{activated}
			label={getDropzoneLabel(handler)}
		/>
	{/snippet}

	<div class="uncommitted-changes-wrap">
		{#if mode === 'unassigned' || changes.current.length > 0}
			<div
				role="presentation"
				data-testid={TestId.UncommittedChanges_Header}
				class="worktree-header"
				class:sticked-top={!scrollTopIsVisible}
			>
				{#if !isCommitting && mode === 'unassigned' && !unassignedSidebaFolded.current}
					<div class="worktree-header__fold">
						<UnassignedFoldButton active={false} onclick={foldUnnassignedView} />
					</div>
				{/if}

				<div class="worktree-header__general">
					{#if isCommitting}
						<WorktreeChangesSelectAll {stackId} />
					{/if}
					<div class="worktree-header__title truncate">
						<h3 class="text-14 text-semibold truncate">{title}</h3>
						<Badge>{changes.current.length}</Badge>
					</div>
				</div>
				{#if changes.current.length > 0}
					<FileListMode bind:mode={listMode} persist="uncommitted" />
				{/if}
			</div>
		{/if}

		{#if changes.current.length > 0}
			<ScrollableContainer
				{onscrollexists}
				autoScroll={false}
				onscrollTop={(visible) => {
					scrollTopIsVisible = visible;
				}}
			>
				{@render fileList()}
			</ScrollableContainer>
		{:else}
			{@render emptyPlaceholder?.()}
		{/if}
	</div>
</Dropzone>

<style>
	.uncommitted-changes-wrap {
		display: flex;
		flex-direction: column;
		height: 100%;
		overflow: hidden;
		background-color: var(--clr-bg-1);
	}

	.worktree-header {
		display: flex;
		align-items: center;
		width: 100%;
		height: 42px;
		padding: 10px 10px 10px 14px;
		gap: 6px;
		border-bottom: 1px solid transparent;
		background-color: var(--clr-bg-1);
		text-wrap: nowrap;
		white-space: nowrap;
	}

	.worktree-header__general {
		display: flex;
		flex: 1;
		align-items: center;
		overflow: hidden;
		gap: 10px;
	}

	.worktree-header__fold {
		display: flex;
		/* Align this icon's position with the folded one.
   	Prevent any position shifting or jumping. */
		margin-left: -3px;
	}

	.worktree-header__title {
		display: flex;
		align-items: center;
		gap: 6px;
	}

	.uncommitted-changes {
		display: block;
	}

	/* MODIFIERS */
	.sticked-top {
		border-bottom-color: var(--clr-border-2);
	}
</style>
