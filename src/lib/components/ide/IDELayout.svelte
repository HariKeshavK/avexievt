<script lang="ts">
	import { onMount, getContext } from 'svelte';
	import { toast } from 'svelte-sonner';
	import { terminalServers, selectedTerminalId, settings } from '$lib/stores';
	import { getCwd, readFile } from '$lib/apis/terminal';
	import ResizableSidePanel from '$lib/components/common/ResizableSidePanel.svelte';
	import IDEFileTree from './IDEFileTree.svelte';
	import IDEEditor from './IDEEditor.svelte';
	import IDETerminalPanel from './IDETerminalPanel.svelte';
	import IDEAssistantPanel from './IDEAssistantPanel.svelte';
	import type { IdeTab } from './IDETabBar.svelte';

	const i18n: any = getContext('i18n');

	let fileTreeOpen = true;
	let fileTreeWidth = 240;
	let terminalPanelExpanded = true;
	let assistantOpen = true;
	let assistantWidth = 320;

	let rootPath = '/';
	let tabs: IdeTab[] = [];
	let activeTabPath: string | null = null;
	let fileContents: Record<string, string> = {};

	// Resolve the active terminal server
	$: activeTerminal = (() => {
		const servers = $terminalServers ?? [];
		if ($selectedTerminalId) {
			const found = servers.find((s: any) => s.id === $selectedTerminalId || s.url === $selectedTerminalId);
			if (found) return found;
		}
		if (servers.length > 0) return servers[0];
		// Direct terminal servers fallback
		const direct = ($settings?.terminalServers ?? []).filter((s: any) => s.url);
		if (direct.length > 0) {
			return {
				id: direct[0].url,
				url: direct[0].url,
				key: direct[0].key ?? '',
				name: direct[0].name ?? 'Terminal'
			};
		}
		// Mock terminal for UI preview without backend
		return {
			id: 'mock',
			url: 'http://localhost:8080/terminal',
			key: 'mock-key',
			name: 'Preview Terminal'
		};
	})();

	$: terminalUrl = activeTerminal?.url ?? '';
	$: terminalKey = activeTerminal?.key ?? localStorage.getItem('token') ?? '';

	const initCwd = async () => {
		if (!terminalUrl) return;
		try {
			const cwdData = await getCwd(terminalUrl, terminalKey);
			if (cwdData?.cwd) {
				rootPath = cwdData.cwd;
			}
		} catch (err) {
			console.error('Failed to get terminal cwd:', err);
		}
	};

	const handleOpenFile = async ({ detail }: CustomEvent<{ path: string; name: string }>) => {
		const { path, name } = detail;
		const existing = tabs.find((t) => t.path === path);
		if (existing) {
			activeTabPath = path;
			return;
		}

		try {
			const content = await readFile(terminalUrl, terminalKey, path);
			if (content === null) {
				toast.error($i18n.t('Failed to read file {{name}}', { name }));
				return;
			}

			fileContents = { ...fileContents, [path]: content };
			tabs = [...tabs, { path, name, dirty: false }];
			activeTabPath = path;
		} catch (err) {
			toast.error($i18n.t('Failed to open file {{name}}', { name }));
		}
	};

	const handleApplyCodeToEditor = (newCode: string) => {
		if (activeTabPath !== null) {
			fileContents = { ...fileContents, [activeTabPath]: newCode };
			tabs = tabs.map((t) => (t.path === activeTabPath ? { ...t, dirty: true } : t));
		}
	};

	let terminalPanelRef: IDETerminalPanel;
	let editorRef: IDEEditor;

	const getRunCommand = (filePath: string): string | null => {
		const ext = filePath.split('.').pop()?.toLowerCase();
		switch (ext) {
			case 'py':
				return `python "${filePath}"`;
			case 'js':
			case 'mjs':
			case 'cjs':
				return `node "${filePath}"`;
			case 'ts':
				return `npx tsx "${filePath}"`;
			case 'sh':
			case 'bash':
				return `bash "${filePath}"`;
			case 'rb':
				return `ruby "${filePath}"`;
			case 'php':
				return `php "${filePath}"`;
			default:
				return null;
		}
	};

	const handleRunActiveFile = async () => {
		if (!activeTabPath) return;
		const cmd = getRunCommand(activeTabPath);
		if (cmd) {
			await editorRef?.saveActiveFile();
			terminalPanelExpanded = true;
			terminalPanelRef?.runCommand(cmd);
			toast.info(`Running: ${cmd}`);
		}
	};

	$: activeFileRunCommand = activeTabPath ? getRunCommand(activeTabPath) : null;

	$: if (terminalUrl) {
		initCwd();
	}

	onMount(() => {
		if (terminalUrl) {
			initCwd();
		}
	});
</script>

{#if !activeTerminal}
	<div class="h-full flex flex-col items-center justify-center p-6 text-center">
		<div class="max-w-md space-y-3">
			<div class="size-12 rounded-2xl bg-gray-100 dark:bg-gray-800 flex items-center justify-center mx-auto text-gray-500">
				<svg class="size-6" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
					<polyline points="4 17 10 11 4 5" />
					<line x1="12" y1="19" x2="20" y2="19" />
				</svg>
			</div>
			<h3 class="text-base font-medium text-gray-800 dark:text-gray-200">
				{$i18n.t('No Terminal Server Connected')}
			</h3>
			<p class="text-xs text-gray-500 dark:text-gray-400">
				{$i18n.t('To use the IDE and file workspace, connect a terminal server or start the open-terminal Docker service in your stack.')}
			</p>
		</div>
	</div>
{:else}
	<div class="flex flex-col h-full w-full min-w-0 overflow-hidden bg-white dark:bg-gray-900">
		<!-- Top IDE Status & Control Bar -->
		<div class="flex items-center justify-between px-3 py-1.5 border-b border-gray-100 dark:border-gray-850 bg-gray-50/80 dark:bg-gray-900/80 text-xs shrink-0 select-none">
			<div class="flex items-center gap-2 min-w-0">
				<button
					type="button"
					class="p-1 rounded hover:bg-gray-200 dark:hover:bg-gray-800 text-gray-500 hover:text-gray-700 dark:hover:text-gray-300 transition {fileTreeOpen ? 'bg-gray-200/60 dark:bg-gray-800/60' : ''}"
					title={$i18n.t('Toggle Explorer')}
					on:click={() => (fileTreeOpen = !fileTreeOpen)}
				>
					<svg class="size-3.5" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
						<rect x="3" y="3" width="18" height="18" rx="2" ry="2"/>
						<line x1="9" y1="3" x2="9" y2="21"/>
					</svg>
				</button>

				<div class="flex items-center gap-1 text-[11px] text-gray-500 dark:text-gray-400 truncate">
					<span class="font-medium text-gray-700 dark:text-gray-300">📁 {rootPath}</span>
				</div>
			</div>

			<div class="flex items-center gap-1.5">
				<!-- Run File Button (when runnable file is open) -->
				{#if activeFileRunCommand}
					<button
						type="button"
						class="flex items-center gap-1 px-2 py-0.5 rounded text-[11px] font-medium bg-emerald-100 dark:bg-emerald-900/40 text-emerald-700 dark:text-emerald-300 hover:bg-emerald-200 dark:hover:bg-emerald-900/60 transition"
						title={`Run: ${activeFileRunCommand}`}
						on:click={handleRunActiveFile}
					>
						<svg class="size-3" viewBox="0 0 24 24" fill="currentColor">
							<polygon points="5 3 19 12 5 21 5 3" />
						</svg>
						<span>{$i18n.t('Run')}</span>
					</button>
				{/if}

				<!-- Terminal Toggle -->
				<button
					type="button"
					class="flex items-center gap-1 px-2 py-0.5 rounded text-[11px] font-medium transition
						{terminalPanelExpanded ? 'bg-gray-200 dark:bg-gray-800 text-gray-800 dark:text-gray-200' : 'text-gray-500 hover:bg-gray-100 dark:hover:bg-gray-800/50'}"
					on:click={() => (terminalPanelExpanded = !terminalPanelExpanded)}
				>
					<svg class="size-3" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
						<polyline points="4 17 10 11 4 5" />
						<line x1="12" y1="19" x2="20" y2="19" />
					</svg>
					<span>{$i18n.t('Terminal')}</span>
				</button>

				<!-- AI Agent Copilot Toggle -->
				<button
					type="button"
					class="flex items-center gap-1 px-2 py-0.5 rounded text-[11px] font-medium transition
						{assistantOpen ? 'bg-purple-100 dark:bg-purple-900/40 text-purple-700 dark:text-purple-300' : 'text-purple-600 dark:text-purple-400 hover:bg-purple-50 dark:hover:bg-purple-950/30'}"
					on:click={() => (assistantOpen = !assistantOpen)}
				>
					<svg class="size-3" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
						<path d="M12 2l3.09 6.26L22 9.27l-5 4.87 1.18 6.88L12 17.77l-6.18 3.25L7 14.14 2 9.27l6.91-1.01L12 2z"/>
					</svg>
					<span>{$i18n.t('AI Agent')}</span>
				</button>
			</div>
		</div>

		<!-- Main IDE Body (Explorer | Editor | AI Assistant) -->
		<div class="flex flex-1 min-h-0 w-full overflow-hidden">
			<!-- Resizable File Tree Explorer -->
			<ResizableSidePanel
				bind:open={fileTreeOpen}
				side="left"
				bind:width={fileTreeWidth}
				minWidth={160}
				maxWidth={400}
				storageKey="ide:fileTreeWidth"
				className="h-full"
			>
				<IDEFileTree
					{terminalUrl}
					{terminalKey}
					{rootPath}
					{activeTabPath}
					on:open={handleOpenFile}
				/>
			</ResizableSidePanel>

			<!-- Center Editor Pane -->
			<div class="flex-1 min-w-0 h-full flex flex-col overflow-hidden">
				<IDEEditor
					bind:this={editorRef}
					{terminalUrl}
					{terminalKey}
					bind:tabs
					bind:activeTabPath
					bind:fileContents
				/>
			</div>

			<!-- Resizable AI Assistant Copilot Panel -->
			<ResizableSidePanel
				bind:open={assistantOpen}
				side="right"
				bind:width={assistantWidth}
				minWidth={240}
				maxWidth={550}
				storageKey="ide:assistantWidth"
				className="h-full"
			>
				<IDEAssistantPanel
					bind:open={assistantOpen}
					{activeFilePath}
					activeFileContent={activeTabPath !== null ? (fileContents[activeTabPath] ?? '') : ''}
					onApplyCode={handleApplyCodeToEditor}
				/>
			</ResizableSidePanel>
		</div>

		<!-- Embedded Terminal Panel at Bottom -->
		<IDETerminalPanel
			bind:this={terminalPanelRef}
			bind:expanded={terminalPanelExpanded}
			height={200}
		/>
	</div>
{/if}
