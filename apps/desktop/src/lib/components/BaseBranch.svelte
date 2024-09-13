<script lang="ts">
	import { Project } from '$lib/backend/projects';
	import CommitAction from '$lib/commit/CommitAction.svelte';
	import CommitCard from '$lib/commit/CommitCard.svelte';
	import { transformAnyCommit } from '$lib/commitLines/transformers';
	import UpdateBaseButton from '$lib/components/UpdateBaseButton.svelte';
	import { projectMergeUpstreamWarningDismissed } from '$lib/config/config';
	import { getGitHost } from '$lib/gitHost/interface/gitHost';
	import { ModeService } from '$lib/modes/service';
	import { showInfo } from '$lib/notifications/toasts';
	import InfoMessage from '$lib/shared/InfoMessage.svelte';
	import { groupByCondition } from '$lib/utils/array';
	import { getContext } from '$lib/utils/context';
	import { BranchController } from '$lib/vbranches/branchController';
	import Button from '@gitbutler/ui/Button.svelte';
	import Checkbox from '@gitbutler/ui/Checkbox.svelte';
	import Modal from '@gitbutler/ui/Modal.svelte';
	import LineGroup from '@gitbutler/ui/commitLines/LineGroup.svelte';
	import { LineManagerFactory, LineSpacer } from '@gitbutler/ui/commitLines/lineManager';
	import type { BaseBranch } from '$lib/baseBranch/baseBranch';

	interface Props {
		base: BaseBranch;
	}

	const { base }: Props = $props();

	const branchController = getContext(BranchController);
	const modeService = getContext(ModeService);
	const gitHost = getGitHost();
	const project = getContext(Project);
	const lineManagerFactory = getContext(LineManagerFactory);

	const mode = $derived(modeService.mode);
	const mergeUpstreamWarningDismissed = $derived(
		projectMergeUpstreamWarningDismissed(branchController.projectId)
	);

	let updateTargetModal = $state<Modal>();
	let mergeUpstreamWarningDismissedCheckbox = $state<boolean>(false);
	let updateBaseButton = $state<UpdateBaseButton | undefined>();

	const { satisfied: commitsAhead, rest: localAndRemoteCommits } = $derived(
		groupByCondition(base.recentCommits, (c) => base.divergedAhead.includes(c.id))
	);

	const multiple = $derived(
		base ? base.upstreamCommits.length > 1 || base.upstreamCommits.length === 0 : false
	);

	const mappedRemoteCommits = $derived(
		base.upstreamCommits.length > 0
			? [...base.upstreamCommits.map(transformAnyCommit), { id: LineSpacer.Remote }]
			: []
	);

	const mappedLocalCommits = $derived.by(() => {
		if (!base.diverged) return [];
		return commitsAhead.length > 0
			? [...commitsAhead.map(transformAnyCommit), { id: LineSpacer.Local }]
			: [];
	});

	const mappedLocalAndRemoteCommits = $derived.by(() => {
		return localAndRemoteCommits.length > 0
			? [...localAndRemoteCommits.map(transformAnyCommit), { id: LineSpacer.LocalAndRemote }]
			: [];
	});

	const lineManager = $derived(
		lineManagerFactory.build(
			{
				remoteCommits: mappedRemoteCommits,
				localCommits: mappedLocalCommits,
				localAndRemoteCommits: mappedLocalAndRemoteCommits,
				integratedCommits: []
			},
			true
		)
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
	</div>
{/if}

{#if !base.diverged && base.upstreamCommits.length > 0}
	<div class="header-wrapper">
		<div class="info-text text-13">
			There {multiple ? 'are' : 'is'}
			{base.upstreamCommits.length} unmerged upstream
			{multiple ? 'commits' : 'commit'}
		</div>

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
	</div>
{/if}

<div class="wrapper">
	<!-- UPSTREAM COMMITS -->
	{#if base.upstreamCommits?.length > 0}
		<div>
			{#each base.upstreamCommits as commit, index}
				<CommitCard
					{commit}
					first={index === 0}
					last={index === base.upstreamCommits.length - 1}
					isUnapplied={true}
					commitUrl={$gitHost?.commitUrl(commit.id)}
					type="remote"
				>
					{#snippet lines(topHeightPx)}
						<LineGroup lineGroup={lineManager.get(commit.id)} {topHeightPx} />
					{/snippet}
				</CommitCard>
			{/each}
		</div>

		{#if base.diverged}
			<CommitAction backgroundColor={false}>
				{#snippet lines()}
					<LineGroup lineGroup={lineManager.get(LineSpacer.Remote)} topHeightPx={0} />
				{/snippet}
				{#snippet action()}
					<Button
						style="warning"
						icon="warning"
						kind="solid"
						tooltip="Resets the target branch to the upstream branch. Will lose all the changes ahead of the upstream branch."
						disabled={$mode?.type !== 'OpenWorkspace'}
						onclick={updateBaseBranch}
					>
						Reset to remote
					</Button>
				{/snippet}
			</CommitAction>
		{/if}
	{/if}

	<!-- DIVERGED (LOCAL) COMMITS -->
	{#if commitsAhead.length > 0}
		<div>
			{#each commitsAhead as commit, index}
				<CommitCard
					{commit}
					first={index === 0}
					last={index === commitsAhead.length - 1}
					isUnapplied={true}
					commitUrl={$gitHost?.commitUrl(commit.id)}
					type="local"
				>
					{#snippet lines(topHeightPx)}
						<LineGroup lineGroup={lineManager.get(commit.id)} {topHeightPx} />
					{/snippet}
				</CommitCard>
			{/each}
		</div>

		<CommitAction backgroundColor={false}>
			{#snippet lines()}
				<LineGroup lineGroup={lineManager.get(LineSpacer.Local)} topHeightPx={0} />
			{/snippet}
			{#snippet action()}
				<Button
					style="pop"
					icon="warning"
					kind="solid"
					tooltip="Resets the upstream branch to the local target branch. Will lose all the remote changes not in the local branch."
					disabled={$mode?.type !== 'OpenWorkspace'}
					onclick={updateBaseBranch}
				>
					Reset to local
				</Button>
			{/snippet}
		</CommitAction>
	{/if}

	<!-- LOCAL AND REMOTE COMMITS -->
	<div>
		{#each localAndRemoteCommits as commit, index}
			<CommitCard
				{commit}
				first={index === 0}
				last={index === localAndRemoteCommits.length - 1}
				isUnapplied={true}
				commitUrl={$gitHost?.commitUrl(commit.id)}
				type="localAndRemote"
			>
				{#snippet lines(topHeightPx)}
					<LineGroup lineGroup={lineManager.get(commit.id)} {topHeightPx} />
				{/snippet}
			</CommitCard>
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
	.header-wrapper {
		display: flex;
		flex-direction: column;
		gap: 16px;
	}
	.message-wrapper {
		display: flex;
		flex-direction: column;
		margin-bottom: 20px;
		gap: 16px;
	}
	.wrapper {
		display: flex;
		flex-direction: column;
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
