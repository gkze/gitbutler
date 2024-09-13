<script lang="ts">
	import Spacer from '../shared/Spacer.svelte';
	import { Project } from '$lib/backend/projects';
	import CommitCard from '$lib/commit/CommitCard.svelte';
	import UpdateBaseButton from '$lib/components/UpdateBaseButton.svelte';
	import { projectMergeUpstreamWarningDismissed } from '$lib/config/config';
	import { getGitHost } from '$lib/gitHost/interface/gitHost';
	import { ModeService } from '$lib/modes/service';
	import { showInfo } from '$lib/notifications/toasts';
	import InfoMessage from '$lib/shared/InfoMessage.svelte';
	import { getContext } from '$lib/utils/context';
	import { BranchController } from '$lib/vbranches/branchController';
	import Button from '@gitbutler/ui/Button.svelte';
	import Checkbox from '@gitbutler/ui/Checkbox.svelte';
	import Modal from '@gitbutler/ui/Modal.svelte';
	import Tooltip from '@gitbutler/ui/Tooltip.svelte';
	import type { BaseBranch } from '$lib/baseBranch/baseBranch';

	interface Props {
		base: BaseBranch;
	}

	const { base }: Props = $props();

	const branchController = getContext(BranchController);
	const modeService = getContext(ModeService);
	const gitHost = getGitHost();
	const project = getContext(Project);

	const mode = $derived(modeService.mode);
	const mergeUpstreamWarningDismissed = $derived(
		projectMergeUpstreamWarningDismissed(branchController.projectId)
	);

	let updateTargetModal = $state<Modal>();
	let mergeUpstreamWarningDismissedCheckbox = $state<boolean>(false);
	let updateBaseButton = $state<UpdateBaseButton | undefined>();

	const multiple = $derived(
		base ? base.upstreamCommits.length > 1 || base.upstreamCommits.length === 0 : false
	);

	async function updateBaseBranch() {
		const infoText = await branchController.updateBaseBranch();
		if (infoText) {
			showInfo('Stashed conflicting branches', infoText);
		}
	}
</script>

{#if base.diverged}
	<div class="message-wrapper">
		<InfoMessage style="warning" filled outlined={false}>
			<svelte:fragment slot="content">
				Your target branch has diverged from upstream.
				<br />
				Target branch is
				<b>
					{`ahead by ${base.divergedAhead.length}`}
				</b>
				commits and
				<b>
					{`behind by ${base.divergedBehind.length}`}
				</b>
				commits
			</svelte:fragment>
		</InfoMessage>

		<Button
			style="error"
			kind="solid"
			tooltip="Resets the target branch to the upstream branch. Will lose all the changes ahead of the upstream branch."
			disabled={$mode?.type !== 'OpenWorkspace'}
			onclick={updateBaseBranch}
		>
			Reset to target to upstream
		</Button>
	</div>
{/if}

<div class="wrapper">
	<div class="info-text text-13">
		There {multiple ? 'are' : 'is'}
		{base.upstreamCommits.length} unmerged upstream
		{multiple ? 'commits' : 'commit'}
	</div>

	{#if base.upstreamCommits?.length > 0 && !base.diverged}
		<UpdateBaseButton bind:this={updateBaseButton} showButton={false} />
		<Button
			style="pop"
			kind="solid"
			tooltip={`Merges the commits from ${base.branchName} into the base of all applied virtual branches`}
			disabled={$mode?.type !== 'OpenWorkspace'}
			onclick={() => {
				if (project.succeedingRebases) {
					updateBaseButton?.openModal();
				} else {
					if ($mergeUpstreamWarningDismissed) {
						updateBaseBranch();
					} else {
						updateTargetModal?.show();
					}
				}
			}}
		>
			Merge into common base
		</Button>
		<div>
			{#each base.upstreamCommits as commit, index}
				<CommitCard
					{commit}
					first={index === 0}
					last={index === base.upstreamCommits.length - 1}
					isUnapplied={true}
					commitUrl={$gitHost?.commitUrl(commit.id)}
					type="remote"
				/>
			{/each}
		</div>
		<Spacer margin={2} />
	{/if}

	<div>
		<Tooltip text="Current base for virtual branches.">
			<h1 class="text-13 info-text text-bold">Local</h1>
		</Tooltip>
		{#each base.recentCommits as commit, index}
			<CommitCard
				{commit}
				first={index === 0}
				last={index === base.recentCommits.length - 1}
				isUnapplied={true}
				commitUrl={$gitHost?.commitUrl(commit.id)}
				type="localAndRemote"
				divergent={base.divergedAhead.includes(commit.id)}
			/>
		{/each}
	</div>
</div>

<Modal
	width="small"
	bind:this={updateTargetModal}
	title="Merge Upstream Work"
	onSubmit={(close) => {
		updateBaseBranch();
		if (mergeUpstreamWarningDismissedCheckbox) {
			mergeUpstreamWarningDismissed.set(true);
		}
		close();
	}}
>
	<div class="modal-content">
		<p class="text-14 text-body">You are about to merge upstream work from your base branch.</p>
	</div>
	<div class="modal-content">
		<h4 class="text-14 text-body text-semibold">What will this do?</h4>
		<p class="modal__small-text text-12 text-body">
			We will try to merge the work that is upstream into each of your virtual branches, so that
			they are all up to date.
		</p>
		<p class="modal__small-text text-12 text-body">
			Any virtual branches that we can't merge cleanly, we will unapply and mark with a blue dot.
			You can merge these manually later.
		</p>
		<p class="modal__small-text text-12 text-body">
			Any virtual branches that are fully integrated upstream will be automatically removed.
		</p>
	</div>

	<label class="modal__dont-show-again" for="dont-show-again">
		<Checkbox name="dont-show-again" bind:checked={mergeUpstreamWarningDismissedCheckbox} />
		<span class="text-12">Don't show this again</span>
	</label>

	{#snippet controls(close)}
		<Button style="ghost" outline onclick={close}>Cancel</Button>
		<Button style="pop" kind="solid" type="submit">Merge Upstream</Button>
	{/snippet}
</Modal>

<style>
	.message-wrapper {
		display: flex;
		flex-direction: column;
		margin-bottom: 20px;
		gap: 16px;
	}
	.wrapper {
		display: flex;
		flex-direction: column;
		gap: 16px;
	}

	.info-text {
		opacity: 0.5;
	}

	.modal-content {
		display: flex;
		flex-direction: column;
		gap: 10px;
		margin-bottom: 20px;

		&:last-child {
			margin-bottom: 0;
		}
	}

	.modal__small-text {
		opacity: 0.6;
	}

	.modal__dont-show-again {
		display: flex;
		align-items: center;
		gap: 8px;
		padding: 14px;
		background-color: var(--clr-bg-2);
		border-radius: var(--radius-m);
		margin-bottom: 6px;
	}
</style>
