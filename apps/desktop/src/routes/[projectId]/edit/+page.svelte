<script lang="ts">
	import { goto } from '$app/navigation';
	import EditMode from '$components/EditMode.svelte';
	import { ModeService, type EditModeMetadata } from '$lib/mode/modeService';
	import { Project } from '$lib/project/project';
	import { getContext } from '@gitbutler/shared/context';

	const modeService = getContext(ModeService);
	const project = getContext(Project);
	const mode = modeService.mode;

	let editModeMetadata = $state<EditModeMetadata>();

	$effect(() => {
		if ($mode?.type === 'Edit') {
			editModeMetadata = $mode.subject;
		} else {
			goto(`/${project.id}`);
		}
	});
</script>

{#if editModeMetadata}
	<EditMode {editModeMetadata} />
{/if}
