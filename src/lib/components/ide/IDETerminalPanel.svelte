<script lang="ts">
	import { getContext } from 'svelte';
	import XTerminal from '$lib/components/chat/XTerminal.svelte';

	const i18n: any = getContext('i18n');

	export let expanded = true;
	export let height = 220; // in px
	export let minHeight = 100;
	export let maxHeight = 600;

	let connected = false;
	let connecting = false;
	let isDragging = false;
	let startY = 0;
	let startHeight = 0;
	let terminalRef: XTerminal;

	export const runCommand = (cmd: string) => {
		if (!expanded) {
			expanded = true;
		}
		terminalRef?.runCommand(cmd);
	};

	const toggle = () => {
		expanded = !expanded;
	};

	const onMouseDown = (e: MouseEvent) => {
		if (!expanded) return;
		isDragging = true;
		startY = e.clientY;
		startHeight = height;
		document.body.style.userSelect = 'none';

		const onMouseMove = (ev: MouseEvent) => {
			const delta = startY - ev.clientY;
			height = Math.max(minHeight, Math.min(maxHeight, startHeight + delta));
		};

		const onMouseUp = () => {
			isDragging = false;
			document.body.style.userSelect = '';
			window.removeEventListener('mousemove', onMouseMove);
			window.removeEventListener('mouseup', onMouseUp);
		};

		window.addEventListener('mousemove', onMouseMove);
		window.addEventListener('mouseup', onMouseUp);
	};
</script>

<div class="flex flex-col border-t border-gray-100 dark:border-gray-850 bg-white dark:bg-gray-900 shrink-0">
	<!-- Resizer handle -->
	{#if expanded}
		<!-- svelte-ignore a11y_no_noninteractive_element_interactions -->
		<div
			class="h-1 -mt-0.5 cursor-row-resize bg-transparent hover:bg-blue-500/50 transition-colors z-10"
			on:mousedown={onMouseDown}
			role="separator"
			tabindex="0"
			aria-label="Resize terminal"
		/>
	{/if}

	<!-- Header bar -->
	<div class="flex items-center justify-between px-3 py-1.5 bg-gray-50 dark:bg-gray-850/60 select-none text-xs">
		<button
			type="button"
			class="flex items-center gap-2 font-medium text-gray-700 dark:text-gray-200 hover:text-gray-900 dark:hover:text-white"
			on:click={toggle}
		>
			<div class="flex items-center gap-1.5">
				<svg class="size-3.5" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
					<polyline points="4 17 10 11 4 5" />
					<line x1="12" y1="19" x2="20" y2="19" />
				</svg>
				<span class="uppercase tracking-wider text-[11px]">{$i18n.t('Terminal')}</span>
			</div>

			<!-- Status indicator -->
			<div class="flex items-center gap-1 text-[11px] font-normal text-gray-400">
				{#if connecting}
					<span class="size-1.5 rounded-full bg-amber-400 animate-pulse" />
					<span>{$i18n.t('Connecting...')}</span>
				{:else if connected}
					<span class="size-1.5 rounded-full bg-emerald-500" />
					<span>{$i18n.t('Connected')}</span>
				{:else}
					<span class="size-1.5 rounded-full bg-gray-400" />
					<span>{$i18n.t('Disconnected')}</span>
				{/if}
			</div>
		</button>

		<div class="flex items-center gap-1">
			<button
				type="button"
				class="p-1 rounded hover:bg-gray-200 dark:hover:bg-gray-700 text-gray-400 hover:text-gray-600 dark:hover:text-gray-300 transition"
				on:click={toggle}
				title={expanded ? $i18n.t('Collapse') : $i18n.t('Expand')}
			>
				<svg class="size-3.5 transition-transform duration-100 {expanded ? 'rotate-180' : ''}" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
					<polyline points="6 9 12 15 18 9" />
				</svg>
			</button>
		</div>
	</div>

	<!-- Terminal Viewport -->
	{#if expanded}
		<div style="height: {height}px;" class="min-h-0 w-full overflow-hidden bg-[#1e1e1e]">
			<XTerminal
				bind:this={terminalRef}
				overlay={isDragging}
				bind:connected
				bind:connecting
			/>
		</div>
	{/if}
</div>
