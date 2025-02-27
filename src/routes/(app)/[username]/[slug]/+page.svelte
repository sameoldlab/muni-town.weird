<script lang="ts">
	// TODO: Investigate deduplicating code between this and the profile page editor.

	import { env } from '$env/dynamic/public';
	import InlineTextEditor from '$lib/components/editors/InlineTextEditor.svelte';
	import { type SvelteComponent } from 'svelte';
	import type { ActionData, PageData } from './$types';
	import { quintOut } from 'svelte/easing';
	import { crossfade } from 'svelte/transition';
	import { onNavigate } from '$app/navigation';
	import Icon from '@iconify/svelte';
	import CompositeMarkdownEditor from '$lib/components/editors/CompositeMarkdownEditor.svelte';
	import { page } from '$app/stores';
	import slugify from 'slugify';
	import { SlideToggle } from '@skeletonlabs/skeleton';
	import type { Page, PageSaveReq } from '$lib/pages/types';
	import { LoroDoc } from 'loro-crdt';
	import base64 from 'base64-js';
	import { checkResponse } from '$lib/utils/http';

	const { data, form }: { data: PageData; form: ActionData } = $props();

	let editingState: { editing: boolean; page: Page } = $state({
		editing: false,
		page: data.page
	});
	let previousLoroSnapshot = $state(undefined) as Uint8Array | undefined;

	const slugifiedSlug = $derived(
		slugify(editingState.page.slug || editingState.page.name || 'untitled', {
			strict: true,
			lower: true
		})
	);

	async function startEdit() {
		if (data.profileMatchesUserSession || data.page.wiki) {
			try {
				const resp = await fetch(`/${$page.params.username}/${$page.params.slug}/loroSnapshot`);
				await checkResponse(resp);
				previousLoroSnapshot = new Uint8Array(await resp.arrayBuffer());
			} catch (_) {}
			editingState.page = data.page;
			editingState.editing = true;
		}
	}

	function cancelEdit(e?: Event) {
		if (e) e.preventDefault();
		editingState.editing = false;
		editingState.page = data.page;
	}

	onNavigate(() => cancelEdit());

	const [fadeOut, fadeIn] = crossfade({
		duration: 500,
		easing: quintOut
	});

	let formDataInput: HTMLInputElement = $state(undefined) as unknown as HTMLInputElement;
	async function handleSubmit() {
		const doc = new LoroDoc();
		doc.setRecordTimestamp(true);
		if (previousLoroSnapshot) doc.import(previousLoroSnapshot);
		const content = doc.getText('content');
		content.delete(0, content.length);
		content.insert(0, editingState.page.markdown);
		const out: PageSaveReq = {
			name: editingState.page.name,
			slug: slugifiedSlug,
			wiki: false,
			loroSnapshot: base64.fromByteArray(doc.export({ mode: 'snapshot' }))
		};
		formDataInput.value = JSON.stringify(out);
	}

	// svelte-ignore non_reactive_update
	let displayNameEditorEl: SvelteComponent;
</script>

<svelte:head>
	<title>
		{data.page.name} | {data.profile.display_name} |
		{env.PUBLIC_INSTANCE_NAME}
	</title>
</svelte:head>
<main class="flex h-full w-full flex-col items-start font-spacemono">
	<div
		class="card relative flex min-h-fit w-full max-w-[1000px] flex-col justify-start gap-4 rounded-xl p-8 text-xl sm:min-h-full"
	>
		<h1 class="relative mx-[2em] mt-2 self-center text-center font-rubik text-4xl">
			{#if !editingState.editing}
				{data.page.name}
			{:else}
				<InlineTextEditor
					bind:this={displayNameEditorEl}
					bind:content={editingState.page.name as string}
				/>
			{/if}
		</h1>

		<div class="text-center">
			By <a href={`/${$page.params.username}`} class="text-blue-300 underline underline-offset-4">
				{data.profile.display_name}
			</a>.
			{#if data.page.wiki}
				Wiki Page.
			{/if}
			See
			<a
				class="text-blue-300 underline underline-offset-4"
				href={`/${$page.params.username}/${$page.params.slug}/revisions`}>Revisions</a
			>.
		</div>

		{#if editingState.editing}
			<label class="mt-2 flex flex-row items-center gap-2">
				<span class="basis-40">Page Slug</span>
				<div class="flex flex-grow flex-col">
					<input
						class="input border-none bg-transparent"
						placeholder="slug"
						disabled={!data.profileMatchesUserSession}
						bind:value={editingState.page.slug}
					/>
				</div>
			</label>
			<div class="flex justify-end text-sm">
				<SlideToggle
					name="wikiMode"
					size="sm"
					bind:checked={editingState.page.wiki}
					disabled={!data.profileMatchesUserSession}
					active="bg-tertiary-600">Wiki Page: Everyone Can Edit</SlideToggle
				>
			</div>
		{/if}

		{#if form?.error}
			<aside class="alert variant-ghost-error my-2 w-full">
				<div class="alert-message">
					<p>Error saving page: {form.error}</p>
				</div>
			</aside>
		{/if}

		<hr />

		{#if editingState.editing && data.profileMatchesUserSession}
			<form method="post" class=" absolute left-8 top-8">
				<button class="variant-ghost-error btn-icon" title="Delete" name="delete">
					<Icon icon="proicons:delete" />
				</button>
			</form>
		{/if}

		<div class="gap-z absolute right-8 top-8 z-10 flex">
			{#if data.profileMatchesUserSession || data.page.wiki}
				{#if !editingState.editing}
					<button
						class="variant-ghost btn-icon absolute right-0 top-0"
						in:fadeIn={{ key: 'edit-buttons' }}
						out:fadeOut={{ key: 'edit-buttons' }}
						title="Edit"
						onclick={startEdit}
					>
						<Icon icon="raphael:edit" />
					</button>
				{:else}
					<form
						method="post"
						class="absolute right-0 top-0 flex flex-col gap-4"
						in:fadeIn={{ key: 'edit-buttons' }}
						out:fadeOut={{ key: 'edit-buttons' }}
						onsubmit={handleSubmit}
					>
						<input type="hidden" name="data" bind:this={formDataInput} />

						<div class="flex flex-row gap-2">
							<button class="variant-ghost-success btn-icon" title="Save">
								<Icon icon="raphael:check" />
							</button>
							<button class="variant-ghost-error btn-icon" title="Cancel" onclick={cancelEdit}>
								<Icon icon="proicons:cancel" />
							</button>
						</div>
					</form>
				{/if}
			{/if}
		</div>

		<div class="flex flex-col gap-8">
			<div
				class="prose relative mx-auto w-full max-w-[1000px] break-words pt-4 dark:prose-invert prose-a:text-blue-400"
			>
				{#if !editingState.editing}
					<article>
						{@html data.page.html}
					</article>
				{:else}
					<CompositeMarkdownEditor bind:content={editingState.page.markdown} />
				{/if}
			</div>
			<!-- {#if data.page.links.length > 0 || editingState.editing}
				<hr />
				<div>
					<h2 class="mb-3 text-center text-2xl font-bold">Links</h2>
					{#if !editingState.editing}
						<ul class="flex flex-col gap-2">
							{#each data.page.links as link}
								<li>
									<a class="variant-ghost btn" href={link.url}>
										{link.label || link.url}
									</a>
								</li>
							{/each}
						</ul>
					{:else}
						<LinksEditor bind:links={editingState.page.links} />
					{/if}
				</div>
			{/if} -->
		</div>
	</div>
</main>

<style>
	:global(.unfurl) {
		& a.title {
			text-decoration: none;
		}
	}
</style>
