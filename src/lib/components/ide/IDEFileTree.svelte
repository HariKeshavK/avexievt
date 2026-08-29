<script lang="ts">
	import { getContext, onMount, createEventDispatcher, tick } from 'svelte';
	import { toast } from 'svelte-sonner';
	import {
		listFiles,
		createDirectory,
		deleteEntry,
		uploadToTerminal,
		type FileEntry
	} from '$lib/apis/terminal';
	import FileTypeIcon from '$lib/components/chat/FileNav/FileTypeIcon.svelte';
	import Spinner from '$lib/components/common/Spinner.svelte';

	const i18n: any = getContext('i18n');
	const dispatch = createEventDispatcher<{
		open: { path: string; name: string };
	}>();

	export let terminalUrl: string = '';
	export let terminalKey: string = '';
	export let rootPath: string = '/';
	export let activeFilePath: string | null = null;

	let loading = false;
	let entries: FileEntry[] = [];
	let expandedDirs = new Set<string>();
	let dirContentsCache = new Map<string, FileEntry[]>();
	let loadingDirs = new Set<string>();

	let searchQuery = '';
	let creatingFileIn: string | null = null;
	let creatingFolderIn: string | null = null;
	let newEntryName = '';
	let inputEl: HTMLInputElement;

	const normalizePath = (p: string) => p.replace(/\\/g, '/').replace(/\/+/g, '/');

	const cleanName = (name: string) => {
		const norm = normalizePath(name);
		return norm.split('/').filter(Boolean).at(-1) ?? name;
	};

	const sortEntries = (items: FileEntry[]) => {
		return [...items].sort((a, b) => {
			if (a.type !== b.type) return a.type === 'directory' ? -1 : 1;
			return a.name.localeCompare(b.name);
		});
	};

	export const refresh = async () => {
		if (!terminalUrl) return;
		loading = true;
		try {
			const res = await listFiles(terminalUrl, terminalKey, rootPath);
			if (res && res.entries) {
				entries = sortEntries(
					res.entries.map((e) => ({ ...e, name: cleanName(e.name) }))
				);
			} else {
				entries = [];
			}
			// Refresh currently expanded subdirectories
			for (const dir of Array.from(expandedDirs)) {
				await loadDir(dir, false);
			}
		} catch (err) {
			console.error('Error listing root files:', err);
		} finally {
			loading = false;
		}
	};

	const loadDir = async (dirPath: string, toggle = true) => {
		const norm = normalizePath(dirPath);
		if (toggle && expandedDirs.has(norm)) {
			expandedDirs.delete(norm);
			expandedDirs = new Set(expandedDirs);
			return;
		}

		loadingDirs.add(norm);
		loadingDirs = new Set(loadingDirs);

		try {
			const res = await listFiles(terminalUrl, terminalKey, norm);
			if (res && res.entries) {
				const sorted = sortEntries(
					res.entries.map((e) => ({ ...e, name: cleanName(e.name) }))
				);
				dirContentsCache.set(norm, sorted);
				dirContentsCache = new Map(dirContentsCache);
				expandedDirs.add(norm);
				expandedDirs = new Set(expandedDirs);
			}
		} catch (err) {
			console.error(`Error loading dir ${norm}:`, err);
		} finally {
			loadingDirs.delete(norm);
			loadingDirs = new Set(loadingDirs);
		}
	};

	const handleEntryClick = async (entry: FileEntry, parentPath: string) => {
		const fullPath = normalizePath(`${parentPath}/${entry.name}`);
		if (entry.type === 'directory') {
			await loadDir(fullPath);
		} else {
			dispatch('open', { path: fullPath, name: entry.name });
		}
	};

	const startCreateFile = async (parentPath: string) => {
		creatingFolderIn = null;
		creatingFileIn = normalizePath(parentPath);
		newEntryName = '';
		await tick();
		inputEl?.focus();
	};

	const startCreateFolder = async (parentPath: string) => {
		creatingFileIn = null;
		creatingFolderIn = normalizePath(parentPath);
		newEntryName = '';
		await tick();
		inputEl?.focus();
	};

	const cancelCreate = () => {
		creatingFileIn = null;
		creatingFolderIn = null;
		newEntryName = '';
	};

	const submitCreate = async () => {
		const name = newEntryName.trim();
		if (!name) {
			cancelCreate();
			return;
		}

		if (creatingFileIn !== null) {
			const targetDir = creatingFileIn;
			cancelCreate();
			try {
				const emptyFile = new File([''], name, { type: 'text/plain' });
				const res = await uploadToTerminal(terminalUrl, terminalKey, targetDir, emptyFile);
				if (res) {
					toast.success($i18n.t('File created'));
					if (targetDir === rootPath) {
						await refresh();
					} else {
						await loadDir(targetDir, false);
					}
					dispatch('open', { path: normalizePath(`${targetDir}/${name}`), name });
				} else {
					toast.error($i18n.t('Failed to create file'));
				}
			} catch (err) {
				toast.error($i18n.t('Failed to create file'));
			}
		} else if (creatingFolderIn !== null) {
			const targetDir = creatingFolderIn;
			cancelCreate();
			try {
				const fullDirPath = normalizePath(`${targetDir}/${name}`);
				const res = await createDirectory(terminalUrl, terminalKey, fullDirPath);
				if (res) {
					toast.success($i18n.t('Folder created'));
					if (targetDir === rootPath) {
						await refresh();
					} else {
						await loadDir(targetDir, false);
					}
				} else {
					toast.error($i18n.t('Failed to create folder'));
				}
			} catch (err) {
				toast.error($i18n.t('Failed to create folder'));
			}
		}
	};

	const handleDelete = async (entry: FileEntry, parentPath: string, e: MouseEvent) => {
		e.stopPropagation();
		const fullPath = normalizePath(`${parentPath}/${entry.name}`);
		if (!confirm($i18n.t('Delete {{name}}?', { name: entry.name }))) return;

		try {
			const res = await deleteEntry(terminalUrl, terminalKey, fullPath);
			if (res) {
				toast.success($i18n.t('{{name}} deleted', { name: entry.name }));
				if (parentPath === rootPath) {
					await refresh();
				} else {
					await loadDir(parentPath, false);
				}
			} else {
				toast.error($i18n.t('Failed to delete {{name}}', { name: entry.name }));
			}
		} catch (err) {
			toast.error($i18n.t('Failed to delete {{name}}', { name: entry.name }));
		}
	};

	$: if (terminalUrl && rootPath) {
		refresh();
	}

	onMount(() => {
		if (terminalUrl) {
			refresh();
		}
	});
	$: filteredEntries = searchQuery.trim()
		? entries.filter((e) => e.name.toLowerCase().includes(searchQuery.trim().toLowerCase()))
		: entries;
</script>

<div class="flex flex-col h-full bg-gray-50/50 dark:bg-gray-900/50 select-none text-xs border-r border-gray-100 dark:border-gray-850">
	<!-- File tree header toolbar -->
	<div class="flex items-center justify-between px-3 py-2 border-b border-gray-100 dark:border-gray-850 bg-white/50 dark:bg-gray-900/50">
		<div class="font-medium text-gray-700 dark:text-gray-200 uppercase tracking-wider text-[11px] truncate flex items-center gap-1.5">
			<span>{$i18n.t('Explorer')}</span>
		</div>
		<div class="flex items-center gap-1">
			<button
				class="p-1 rounded hover:bg-gray-200 dark:hover:bg-gray-700 text-gray-500 hover:text-gray-700 dark:hover:text-gray-300 transition"
				title={$i18n.t('New File')}
				on:click={() => startCreateFile(rootPath)}
			>
				<svg class="size-3.5" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
					<path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/>
					<polyline points="14 2 14 8 20 8"/>
					<line x1="12" y1="18" x2="12" y2="12"/>
					<line x1="9" y1="15" x2="15" y2="15"/>
				</svg>
			</button>
			<button
				class="p-1 rounded hover:bg-gray-200 dark:hover:bg-gray-700 text-gray-500 hover:text-gray-700 dark:hover:text-gray-300 transition"
				title={$i18n.t('New Folder')}
				on:click={() => startCreateFolder(rootPath)}
			>
				<svg class="size-3.5" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
					<path d="M22 19a2 2 0 0 1-2 2H4a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h5l2 3h9a2 2 0 0 1 2 2z"/>
					<line x1="12" y1="11" x2="12" y2="17"/>
					<line x1="9" y1="14" x2="15" y2="14"/>
				</svg>
			</button>
			<button
				class="p-1 rounded hover:bg-gray-200 dark:hover:bg-gray-700 text-gray-500 hover:text-gray-700 dark:hover:text-gray-300 transition"
				title={$i18n.t('Refresh')}
				on:click={refresh}
			>
				<svg class="size-3.5 {loading ? 'animate-spin' : ''}" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
					<path d="M21.5 2v6h-6M21.34 15.57a10 10 0 1 1-.57-8.38l5.67-5.67"/>
				</svg>
			</button>
		</div>
	</div>

	<!-- Search / Filter input -->
	<div class="px-2 py-1.5 border-b border-gray-100 dark:border-gray-850">
		<div class="flex items-center gap-1.5 px-2 py-1 bg-white dark:bg-gray-800 rounded border border-gray-200/60 dark:border-gray-700/60">
			<svg class="size-3 text-gray-400 shrink-0" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
				<circle cx="11" cy="11" r="8"/>
				<line x1="21" y1="21" x2="16.65" y2="16.65"/>
			</svg>
			<input
				bind:value={searchQuery}
				placeholder={$i18n.t('Filter files...')}
				class="w-full bg-transparent border-none outline-none text-xs text-gray-800 dark:text-gray-200 p-0 placeholder:text-gray-400"
			/>
			{#if searchQuery}
				<button
					type="button"
					class="p-0.5 text-gray-400 hover:text-gray-600 dark:hover:text-gray-200"
					on:click={() => (searchQuery = '')}
				>
					<svg class="size-2.5" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
						<line x1="18" y1="6" x2="6" y2="18"/>
						<line x1="6" y1="6" x2="18" y2="18"/>
					</svg>
				</button>
			{/if}
		</div>
	</div>

	<!-- File list body -->
	<div class="flex-1 overflow-y-auto overflow-x-hidden p-1 space-y-0.5">
		{#if loading && entries.length === 0}
			<div class="flex items-center justify-center p-6 text-gray-400">
				<Spinner className="size-4" />
			</div>
		{:else if filteredEntries.length === 0}
			<div class="text-center py-8 text-gray-400 dark:text-gray-500 text-xs">
				{$i18n.t('No files found')}
			</div>
		{:else}
			{#if creatingFileIn === rootPath || creatingFolderIn === rootPath}
				<div class="flex items-center gap-1.5 px-2 py-1 bg-gray-100 dark:bg-gray-800 rounded">
					<FileTypeIcon name={newEntryName} type={creatingFolderIn ? 'directory' : 'file'} />
					<input
						bind:this={inputEl}
						bind:value={newEntryName}
						class="w-full bg-transparent border-none outline-none text-xs text-gray-800 dark:text-gray-200 p-0"
						placeholder={creatingFolderIn ? 'folder-name' : 'file.ext'}
						on:keydown={(e) => {
							if (e.key === 'Enter') submitCreate();
							if (e.key === 'Escape') cancelCreate();
						}}
						on:blur={submitCreate}
					/>
				</div>
			{/if}

			{#each filteredEntries as entry (entry.name)}
				{@const fullPath = normalizePath(`${rootPath}/${entry.name}`)}
				{@const isExpanded = expandedDirs.has(fullPath)}
				{@const isActive = activeFilePath === fullPath}
				{@const isDirLoading = loadingDirs.has(fullPath)}

				<div>
					<button
						type="button"
						class="w-full group flex items-center justify-between px-2 py-1 rounded text-left hover:bg-gray-100 dark:hover:bg-gray-800/60 transition-colors
							{isActive ? 'bg-blue-50 dark:bg-blue-900/20 text-blue-600 dark:text-blue-400 font-medium' : 'text-gray-700 dark:text-gray-300'}"
						on:click={() => handleEntryClick(entry, rootPath)}
					>
						<div class="flex items-center gap-1.5 min-w-0 flex-1">
							{#if entry.type === 'directory'}
								<svg
									class="size-3 shrink-0 text-gray-400 transition-transform duration-100 {isExpanded ? 'rotate-90' : ''}"
									viewBox="0 0 24 24"
									fill="none"
									stroke="currentColor"
									stroke-width="2"
								>
									<polyline points="9 18 15 12 9 6" />
								</svg>
							{/if}
							<FileTypeIcon name={entry.name} type={entry.type} />
							<span class="truncate">{entry.name}</span>
							{#if isDirLoading}
								<Spinner className="size-2.5 ml-1 text-gray-400" />
							{/if}
						</div>

						<div class="flex items-center opacity-0 group-hover:opacity-100 transition-opacity gap-0.5 ml-1 shrink-0">
							{#if entry.type === 'directory'}
								<span
									role="button"
									tabindex="0"
									class="p-0.5 hover:bg-gray-200 dark:hover:bg-gray-700 rounded text-gray-400 hover:text-gray-600"
									title="New file inside"
									on:click|stopPropagation={() => startCreateFile(fullPath)}
									on:keydown|stopPropagation
								>
									<svg class="size-3" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
										<path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/>
										<line x1="12" y1="18" x2="12" y2="12"/>
										<line x1="9" y1="15" x2="15" y2="15"/>
									</svg>
								</span>
							{/if}
							<span
								role="button"
								tabindex="0"
								class="p-0.5 hover:bg-red-100 dark:hover:bg-red-900/30 rounded text-gray-400 hover:text-red-500"
								title="Delete"
								on:click|stopPropagation={(e) => handleDelete(entry, rootPath, e)}
								on:keydown|stopPropagation
							>
								<svg class="size-3" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
									<polyline points="3 6 5 6 21 6"/>
									<path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"/>
								</svg>
							</span>
						</div>
					</button>

					<!-- Render nested folder children if expanded -->
					{#if entry.type === 'directory' && isExpanded}
						<div class="pl-4 border-l border-gray-200 dark:border-gray-800 ml-3.5 space-y-0.5 my-0.5">
							{#if creatingFileIn === fullPath || creatingFolderIn === fullPath}
								<div class="flex items-center gap-1.5 px-2 py-1 bg-gray-100 dark:bg-gray-800 rounded">
									<FileTypeIcon name={newEntryName} type={creatingFolderIn ? 'directory' : 'file'} />
									<input
										bind:this={inputEl}
										bind:value={newEntryName}
										class="w-full bg-transparent border-none outline-none text-xs text-gray-800 dark:text-gray-200 p-0"
										placeholder={creatingFolderIn ? 'folder-name' : 'file.ext'}
										on:keydown={(e) => {
											if (e.key === 'Enter') submitCreate();
											if (e.key === 'Escape') cancelCreate();
										}}
										on:blur={submitCreate}
									/>
								</div>
							{/if}

							{#each dirContentsCache.get(fullPath) ?? [] as childEntry (childEntry.name)}
								{@const childFullPath = normalizePath(`${fullPath}/${childEntry.name}`)}
								{@const isChildActive = activeFilePath === childFullPath}
								<button
									type="button"
									class="w-full group flex items-center justify-between px-2 py-1 rounded text-left hover:bg-gray-100 dark:hover:bg-gray-800/60 transition-colors
										{isChildActive ? 'bg-blue-50 dark:bg-blue-900/20 text-blue-600 dark:text-blue-400 font-medium' : 'text-gray-700 dark:text-gray-300'}"
									on:click={() => handleEntryClick(childEntry, fullPath)}
								>
									<div class="flex items-center gap-1.5 min-w-0 flex-1">
										<FileTypeIcon name={childEntry.name} type={childEntry.type} />
										<span class="truncate">{childEntry.name}</span>
									</div>
									<div class="flex items-center opacity-0 group-hover:opacity-100 transition-opacity gap-0.5 ml-1 shrink-0">
										<span
											role="button"
											tabindex="0"
											class="p-0.5 hover:bg-red-100 dark:hover:bg-red-900/30 rounded text-gray-400 hover:text-red-500"
											title="Delete"
											on:click|stopPropagation={(e) => handleDelete(childEntry, fullPath, e)}
											on:keydown|stopPropagation
										>
											<svg class="size-3" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
												<polyline points="3 6 5 6 21 6"/>
												<path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"/>
											</svg>
										</span>
									</div>
								</button>
							{/each}
						</div>
					{/if}
				</div>
			{/each}
		{/if}
	</div>
</div>
