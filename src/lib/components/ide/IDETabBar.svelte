<script lang="ts">
	import { createEventDispatcher } from 'svelte';

	export interface IdeTab {
		path: string;
		name: string;
		dirty: boolean;
	}

	export let tabs: IdeTab[] = [];
	export let activeTabPath: string | null = null;

	const dispatch = createEventDispatcher<{
		select: { path: string };
		close: { path: string };
	}>();
</script>

{#if tabs.length > 0}
	<div
		class="flex items-center gap-0 overflow-x-auto scrollbar-none border-b border-gray-100 dark:border-gray-850 bg-white dark:bg-gray-900 shrink-0"
		role="tablist"
	>
		{#each tabs as tab (tab.path)}
			<button
				role="tab"
				aria-selected={tab.path === activeTabPath}
				class="group flex items-center gap-1.5 h-8 px-3 text-xs shrink-0 border-r border-gray-100 dark:border-gray-850 transition-colors duration-75
					{tab.path === activeTabPath
					? 'bg-gray-50 dark:bg-gray-850 text-gray-900 dark:text-gray-100'
					: 'text-gray-400 dark:text-gray-500 hover:bg-gray-50 dark:hover:bg-gray-850 hover:text-gray-700 dark:hover:text-gray-300'}"
				on:click={() => dispatch('select', { path: tab.path })}
			>
				{#if tab.dirty}
					<span class="size-1.5 rounded-full bg-blue-400 dark:bg-blue-500 shrink-0" title="Unsaved changes" />
				{/if}
				<span class="max-w-[140px] truncate">{tab.name}</span>
				<!-- svelte-ignore a11y_click_events_have_key_events -->
				<span
					role="button"
					tabindex="0"
					class="size-4 rounded flex items-center justify-center opacity-0 group-hover:opacity-100 transition-opacity hover:bg-gray-200 dark:hover:bg-gray-700 ml-0.5 shrink-0"
					title="Close"
					on:click|stopPropagation={() => dispatch('close', { path: tab.path })}
					on:keydown={(e) => e.key === 'Enter' && dispatch('close', { path: tab.path })}
				>
					<svg class="size-2.5" viewBox="0 0 10 10" fill="none">
						<path d="M2 2l6 6M8 2l-6 6" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" />
					</svg>
				</span>
			</button>
		{/each}
	</div>
{/if}
