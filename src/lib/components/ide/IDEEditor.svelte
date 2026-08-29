<script lang="ts">
	import { getContext, tick } from 'svelte';
	import { toast } from 'svelte-sonner';
	import { uploadToTerminal } from '$lib/apis/terminal';
	import FileCodeEditor from '$lib/components/chat/FileNav/FileCodeEditor.svelte';
	import IDETabBar from './IDETabBar.svelte';
	import type { IdeTab } from './IDETabBar.svelte';

	const i18n: any = getContext('i18n');

	// Active terminal connection info (passed down from IDELayout)
	export let terminalUrl: string = '';
	export let terminalKey: string = '';

	// Open tabs — exported so IDELayout can push new files in
	export let tabs: IdeTab[] = [];
	export let activeTabPath: string | null = null;

	// Map of path → saved content; dirtyContents holds live editor state
	export let fileContents: Record<string, string> = {};

	let editorRef: FileCodeEditor;
	let saving = false;

	// The value currently bound to the editor
	let editorValue: string = '';

	// Keep a local map of live (possibly unsaved) content per tab
	let liveContents: Record<string, string> = {};

	// When active tab changes: push saved content into editor
	$: if (activeTabPath !== null) {
		const incoming = fileContents[activeTabPath] ?? '';
		liveContents = { ...liveContents, [activeTabPath]: liveContents[activeTabPath] ?? incoming };
		editorValue = liveContents[activeTabPath];
	} else {
		editorValue = '';
	}

	// When the bound editorValue changes, track dirty state
	$: if (activeTabPath !== null) {
		const saved = fileContents[activeTabPath] ?? '';
		const live = editorValue;
		liveContents = { ...liveContents, [activeTabPath]: live };
		tabs = tabs.map((t) =>
			t.path === activeTabPath ? { ...t, dirty: live !== saved } : t
		);
	}

	const handleTabSelect = async ({ detail }: CustomEvent<{ path: string }>) => {
		if (activeTabPath !== null) {
			// Stash current live content before switching
			liveContents = { ...liveContents, [activeTabPath]: editorValue };
		}
		activeTabPath = detail.path;
		await tick();
		editorRef?.setValue(liveContents[activeTabPath] ?? fileContents[activeTabPath] ?? '');
	};

	const handleTabClose = ({ detail }: CustomEvent<{ path: string }>) => {
		const path = detail.path;
		const tab = tabs.find((t) => t.path === path);
		if (tab?.dirty) {
			if (!confirm(`${tab.name} has unsaved changes. Close anyway?`)) return;
		}
		tabs = tabs.filter((t) => t.path !== path);
		const { [path]: _c, ...restC } = fileContents;
		fileContents = restC;
		const { [path]: _l, ...restL } = liveContents;
		liveContents = restL;

		if (activeTabPath === path) {
			activeTabPath = tabs.at(-1)?.path ?? null;
		}
	};

	const doSave = async (path: string, content: string) => {
		if (!terminalUrl || saving) return;
		const fileName = path.split('/').pop() ?? 'file';
		const dir = path.substring(0, path.lastIndexOf('/') + 1) || '/';

		saving = true;
		try {
			const file = new File([content], fileName, { type: 'text/plain' });
			const result = await uploadToTerminal(terminalUrl, terminalKey, dir, file);
			if (result) {
				fileContents = { ...fileContents, [path]: content };
				// Re-mark clean
				tabs = tabs.map((t) => (t.path === path ? { ...t, dirty: false } : t));
				toast.success($i18n.t('File saved'));
			} else {
				toast.error($i18n.t('Failed to save file'));
			}
		} finally {
			saving = false;
		}
	};

	// onSave callback passed to FileCodeEditor (triggered by Mod-S inside editor)
	const handleEditorSave = async (content: string) => {
		if (activeTabPath === null) return;
		await doSave(activeTabPath, content);
	};

	export const saveActiveFile = async (): Promise<boolean> => {
		if (activeTabPath === null) return false;
		const content = liveContents[activeTabPath] ?? editorValue ?? fileContents[activeTabPath] ?? '';
		await doSave(activeTabPath, content);
		return true;
	};
</script>

<div class="flex flex-col h-full min-h-0 overflow-hidden">
	<IDETabBar {tabs} {activeTabPath} on:select={handleTabSelect} on:close={handleTabClose} />

	<div class="flex-1 min-h-0 overflow-hidden relative">
		{#if activeTabPath !== null}
			{#if saving}
				<div
					class="absolute inset-0 z-10 flex items-center justify-center bg-white/40 dark:bg-gray-900/40 backdrop-blur-[2px]"
				>
					<span class="text-xs text-gray-500 dark:text-gray-400">{$i18n.t('Saving...')}</span>
				</div>
			{/if}
			<FileCodeEditor
				bind:this={editorRef}
				bind:value={editorValue}
				filePath={activeTabPath}
				onSave={handleEditorSave}
			/>
		{:else}
			<div class="h-full flex items-center justify-center">
				<div class="text-center text-xs text-gray-400 dark:text-gray-600 select-none space-y-1">
					<svg
						class="size-8 mx-auto opacity-40 mb-2"
						viewBox="0 0 24 24"
						fill="none"
						stroke="currentColor"
						stroke-width="1"
					>
						<path
							d="M13.5 6L10 18.5M6.5 8.5L3 12l3.5 3.5M17.5 8.5L21 12l-3.5 3.5"
							stroke-linecap="round"
							stroke-linejoin="round"
						/>
					</svg>
					<p>{$i18n.t('Select a file from the tree to edit it')}</p>
					<p class="opacity-60">{$i18n.t('Ctrl+S / ⌘S to save')}</p>
				</div>
			</div>
		{/if}
	</div>
</div>
