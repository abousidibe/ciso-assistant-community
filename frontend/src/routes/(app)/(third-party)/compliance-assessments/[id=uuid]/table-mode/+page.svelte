<script lang="ts">
	import CreateModal from '$lib/components/Modals/CreateModal.svelte';
	import DeleteConfirmModal from '$lib/components/Modals/DeleteConfirmModal.svelte';
	import {
		complianceResultTailwindColorMap,
		complianceStatusTailwindColorMap
	} from '$lib/utils/constants';
	import { getModelInfo } from '$lib/utils/crud';
	import { safeTranslate } from '$lib/utils/i18n';
	import * as m from '$paraglide/messages';
	import {
		Accordion,
		AccordionItem,
		getModalStore,
		RadioGroup,
		RadioItem,
		SlideToggle,
		type ModalComponent,
		type ModalSettings,
		type ModalStore
	} from '@skeletonlabs/skeleton';
	import type { Actions, PageData } from '../../table-mode/$types';
	import { page } from '$app/stores';

	export let data: PageData;
	export let form: Actions;

	/** Is the page used for shallow routing? */
	export let shallow = false;

	export let actionPath: string = '';
	export let questionnaireOnly: boolean = false;
	export let assessmentOnly: boolean = false;
	export let invalidateAll: boolean = true;

	const result_options = [
		{ id: 'not_assessed', label: m.notAssessed() },
		{ id: 'non_compliant', label: m.nonCompliant() },
		{ id: 'partially_compliant', label: m.partiallyCompliant() },
		{ id: 'compliant', label: m.compliant() },
		{ id: 'not_applicable', label: m.notApplicable() }
	];
	const status_options = [
		{ id: 'to_do', label: m.toDo() },
		{ id: 'in_progress', label: m.inProgress() },
		{ id: 'in_review', label: m.inReview() },
		{ id: 'done', label: m.done() }
	];

	const requirementHashmap = Object.fromEntries(
		data.requirements.map((requirement) => [requirement.id, requirement])
	);

	$: createdEvidence = form?.createdEvidence;

	function title(requirementAssessment) {
		const requirement =
			requirementHashmap[requirementAssessment.requirement] ?? requirementAssessment;
		return requirement.display_short ? requirement.display_short : (requirement.name ?? '');
	}

	// Function to update requirement assessments
	function update(requirementAssessment, field: string, value: string, question: string = '') {
		if (question) {
			const questionIndex = requirementAssessment.answer.questions.findIndex(
				(q) => q.urn === question.urn
			);
			requirementAssessment.answer.questions[questionIndex].answer = value;
			value = requirementAssessment.answer;
		}
		const form = document.getElementById(`tableModeForm-${requirementAssessment.id}`);
		const formData = {
			id: requirementAssessment.id,
			[field]: value
		};
		fetch(form.action, {
			method: 'POST',
			body: JSON.stringify(formData)
		});
	}

	function addColor(result: string, map: Record<string, string>) {
		return map[result];
	}

	let questionnaireMode = questionnaireOnly
		? true
		: assessmentOnly
			? false
			: $page.data.user.is_third_party
				? true
				: false;

	const modalStore: ModalStore = getModalStore();

	function modalEvidenceCreateForm(createform): void {
		const modalComponent: ModalComponent = {
			ref: CreateModal,
			props: {
				form: createform,
				formAction: `${actionPath}?/createEvidence`,
				invalidateAll: invalidateAll,
				model: data.evidenceModel,
				debug: false
			}
		};
		const modal: ModalSettings = {
			type: 'component',
			component: modalComponent,
			title: safeTranslate('add-' + data.evidenceModel.localName)
		};
		modalStore.trigger(modal);
	}

	let addedEvidence = 0;

	$: if (createdEvidence && shallow) {
		data.requirements
			.find((requirementAssessment) => requirementAssessment.id === createdEvidence.requirements[0])
			.evidences.push({
				str: createdEvidence.name,
				id: createdEvidence.id
			});
		createdEvidence = undefined;
		addedEvidence = +1;
	}

	function modalConfirmDelete(id: string, name: string): void {
		const modalComponent: ModalComponent = {
			ref: DeleteConfirmModal,
			props: {
				_form: data.deleteForm,
				formAction: `/evidences?/delete`,
				id: id,
				invalidateAll: invalidateAll,
				debug: false,
				URLModel: getModelInfo('evidences').urlModel
			}
		};
		const modal: ModalSettings = {
			type: 'component',
			component: modalComponent,
			// Data
			title: m.deleteModalTitle(),
			body: `${m.deleteModalMessage({ name })}`
		};
		modalStore.trigger(modal);
		data.requirements.forEach((requirementAssessment) => {
			console.log(requirementAssessment.evidences);
			requirementAssessment.evidences = requirementAssessment.evidences.filter(
				(evidence) => evidence.id !== id
			);
		});
	}
</script>

<div class="flex flex-col space-y-4 whitespace-pre-line">
	<div
		class="card px-6 py-4 bg-white flex flex-col justify-between shadow-lg w-full h-full space-y-2"
	>
		{#if !(questionnaireOnly ? !assessmentOnly : assessmentOnly)}
			<div class="sticky top-0 p-2 z-10 card bg-white">
				<div class="flex items-center justify-center space-x-4">
					{#if questionnaireMode}
						<p class="font-bold text-sm">{m.assessmentMode()}</p>
					{:else}
						<p class="font-bold text-sm text-green-500">{m.assessmentMode()}</p>
					{/if}
					<SlideToggle
						name="questionnaireToggle"
						class="flex flex-row items-center justify-center"
						active="bg-primary-500"
						background="bg-green-500"
						bind:checked={questionnaireMode}
						on:click={() => (questionnaireMode = !questionnaireMode)}
					>
						{#if questionnaireMode}
							<p class="font-bold text-sm text-primary-500">{m.questionnaireMode()}</p>
						{:else}
							<p class="font-bold text-sm">{m.questionnaireMode()}</p>
						{/if}
					</SlideToggle>
				</div>
			</div>
		{/if}
		{#each data.requirement_assessments as requirementAssessment}
			<div class="w-2"></div>

			<span class="relative flex justify-center py-4">
				<div
					class="absolute inset-x-0 top-1/2 h-px -translate-y-1/2 bg-transparent bg-gradient-to-r from-transparent via-gray-500 to-transparent opacity-75"
				></div>

				<span class="relative z-10 bg-white px-6 text-orange-600 font-semibold text-xl z-auto">
					{title(requirementAssessment)}
				</span>
			</span>
			<div class="h-2"></div>
			<div
				class="flex flex-col items-center justify-center border px-4 py-2 shadow rounded-xl space-y-2"
			>
				{#if requirementAssessment.description}
					<div class="flex w-full font-semibold">
						{requirementAssessment.description}
					</div>
				{/if}
				{#if requirementAssessment.assessable}
					<form
						class="flex flex-col space-y-2 items-center justify-evenly w-full"
						id="tableModeForm-{requirementAssessment.id}"
						action="{actionPath}?/updateRequirementAssessment"
						method="post"
					>
						{#if !questionnaireMode}
							<div class="flex flex-row w-full space-x-2 my-4">
								<div class="flex flex-col items-center w-1/2">
									<p class="flex items-center font-semibold text-blue-600 italic">{m.status()}</p>
									<RadioGroup class="w-full flex-wrap items-center">
										{#each status_options as option}
											<RadioItem
												class="h-full"
												id={option.id}
												active={addColor(
													requirementAssessment.status,
													complianceStatusTailwindColorMap
												)}
												value={option.id}
												bind:group={requirementAssessment.status}
												name="status"
												on:click={() => {
													const newStatus =
														requirementAssessment.status === option.id ? 'to_do' : option.id;
													requirementAssessment.status = newStatus;
													update(requirementAssessment, 'status', newStatus);
												}}>{option.label}</RadioItem
											>
										{/each}
									</RadioGroup>
								</div>
								<div class="flex flex-col items-center w-1/2">
									<p class="flex items-center font-semibold text-purple-600 italic">
										{m.result()}
									</p>
									<RadioGroup class="w-full flex-wrap items-center">
										{#each result_options as option}
											<RadioItem
												class="h-full"
												active={addColor(
													requirementAssessment.result,
													complianceResultTailwindColorMap
												)}
												id={option.id}
												value={option.id}
												bind:group={requirementAssessment.result}
												name="result"
												on:click={() => {
													const newResult =
														requirementAssessment.result === option.id ? 'not_assessed' : option.id;
													requirementAssessment.result = newResult;
													update(requirementAssessment, 'result', newResult); // Update result for both select and deselect
												}}
												>{option.label}
											</RadioItem>
										{/each}
									</RadioGroup>
								</div>
							</div>
						{/if}
						{#if requirementAssessment.answer != null && Object.keys(requirementAssessment.answer).length !== 0}
							<div class="flex flex-col w-full space-y-2">
								{#each requirementAssessment.answer.questions as question}
									<li class="flex flex-col space-y-2 rounded-xl">
										<p>{question.text}</p>
										{#if shallow}
											{#if question.answer}
												<p class="text-primary-500 font-semibold">{question.answer}</p>
											{:else}
												<p class="text-gray-400 italic">{m.noAnswer()}</p>
											{/if}
										{:else if question.type === 'unique_choice'}
											<RadioGroup
												class="flex-col"
												active="variant-filled-primary"
												hover="hover:variant-soft-primary"
											>
												{#each question.options as option}
													<RadioItem
														class="shadow-md"
														bind:group={question.answer}
														name="question"
														value={option}
														on:click={() => {
															const newAnswer = question.answer === option ? null : option;
															question.answer = newAnswer;
															update(requirementAssessment, 'answer', newAnswer, question);
														}}
														>{option}
													</RadioItem>
												{/each}
											</RadioGroup>
										{:else if question.type === 'date'}
											<input
												type="date"
												placeholder=""
												class="input w-fit"
												bind:value={question.answer}
												on:change={() =>
													update(requirementAssessment, 'answer', question.answer, question)}
												{...$$restProps}
											/>
										{:else}
											<textarea
												placeholder=""
												class="input w-full"
												bind:value={question.answer}
												on:keydown={(event) => event.key === 'Enter' && event.preventDefault()}
												on:change={() =>
													update(requirementAssessment, 'answer', question.answer, question)}
												{...$$restProps}
											/>
										{/if}
									</li>
								{/each}
							</div>
						{/if}
						<div class="flex flex-col w-full place-items-center">
							<Accordion regionCaret="flex">
								<AccordionItem caretOpen="rotate-0" caretClosed="-rotate-90">
									<svelte:fragment slot="summary"
										><p class="flex">{m.observation()}</p></svelte:fragment
									>
									<svelte:fragment slot="content">
										<div>
											{#if shallow}
												{#if requirementAssessment.observation}
													<p class="text-primary-500">{requirementAssessment.observation}</p>
												{:else}
													<p class="text-gray-400 italic">{m.noObservation()}</p>
												{/if}
											{:else}
												<textarea
													placeholder=""
													class="input w-full"
													bind:value={requirementAssessment.observation}
													on:keydown={(event) => event.key === 'Enter' && event.preventDefault()}
												/>
												{#if requirementAssessment.observationBuffer !== requirementAssessment.observation}
													<button
														class="rounded-md w-8 h-8 border shadow-lg hover:bg-green-300 hover:text-green-500 duration-300"
														on:click={() => {
															update(
																requirementAssessment,
																'observation',
																requirementAssessment.observation
															);
															requirementAssessment.observationBuffer =
																requirementAssessment.observation;
														}}
														type="button"
													>
														<i class="fa-solid fa-check opacity-70"></i>
													</button>
													<button
														class="rounded-md w-8 h-8 border shadow-lg hover:bg-red-300 hover:text-red-500 duration-300"
														on:click={() =>
															(requirementAssessment.observation =
																requirementAssessment.observationBuffer)}
														type="button"
													>
														<i class="fa-solid fa-xmark opacity-70"></i>
													</button>
												{/if}
											{/if}
										</div>
									</svelte:fragment>
								</AccordionItem>
								<AccordionItem caretOpen="rotate-0" caretClosed="-rotate-90">
									<svelte:fragment slot="summary"
										><p class="flex items-center space-x-2">
											<span>{m.evidence()}</span>
											{#key addedEvidence}
												{#if requirementAssessment.evidences != null}
													<span class="badge variant-soft-primary"
														>{requirementAssessment.evidences.length}</span
													>
												{/if}
											{/key}
										</p></svelte:fragment
									>
									<svelte:fragment slot="content">
										<div class="flex flex-row space-x-2 items-center">
											{#if !shallow}
												<button
													class="btn variant-filled-primary self-start"
													on:click={() =>
														modalEvidenceCreateForm(requirementAssessment.evidenceCreateForm)}
													type="button"><i class="fa-solid fa-plus mr-2" />{m.addEvidence()}</button
												>
											{/if}
											{#key addedEvidence}
												{#each requirementAssessment.evidences as evidence}
													<p class="card p-2">
														<a class="hover:text-primary-500" href="/evidences/{evidence.id}"
															><i class="fa-solid fa-file mr-2"></i>{evidence.str}</a
														>
														{#if !shallow}
															<button
																class="cursor-pointer"
																on:click={(_) => modalConfirmDelete(evidence.id, evidence.str)}
																type="button"
															>
																<i class="fa-solid fa-xmark ml-2 text-red-500"></i>
															</button>
														{/if}
													</p>
												{/each}
											{/key}
										</div>
									</svelte:fragment>
								</AccordionItem>
							</Accordion>
						</div>
					</form>
				{/if}
			</div>
		{/each}
	</div>
</div>
