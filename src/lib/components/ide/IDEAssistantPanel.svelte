<script lang="ts">
	import { getContext, tick } from 'svelte';
	import { toast } from 'svelte-sonner';
	import { models, settings, user } from '$lib/stores';
	import { chatCompletion } from '$lib/apis/openai';
	import { createOpenAITextStream } from '$lib/apis/streaming';
	import { marked } from 'marked';
	import DOMPurify from 'dompurify';
	import Spinner from '$lib/components/common/Spinner.svelte';

	const i18n: any = getContext('i18n');

	export let open = true;
	export let activeFilePath: string | null = null;
	export let activeFileContent: string = '';
	export let onApplyCode: ((code: string) => void) | null = null;

	type Message = {
		role: 'user' | 'assistant' | 'system';
		content: string;
	};

	let messages: Message[] = [];
	let prompt = '';
	let loading = false;
	let abortController: AbortController | null = null;
	let chatContainerEl: HTMLDivElement;

	// Pick default model (prefer Ollama / Qwen if present, or user's default model)
	$: defaultModelId = (() => {
		const allModels = $models ?? [];
		if (allModels.length === 0) return '';
		const qwenModel = allModels.find((m) => m.id.toLowerCase().includes('qwen') || m.name?.toLowerCase().includes('qwen'));
		if (qwenModel) return qwenModel.id;
		const ollamaModel = allModels.find((m) => m.owned_by === 'ollama');
		if (ollamaModel) return ollamaModel.id;
		return allModels[0]?.id ?? '';
	})();

	let selectedModel = '';
	$: if (!selectedModel && defaultModelId) {
		selectedModel = defaultModelId;
	}

	const scrollToBottom = async () => {
		await tick();
		if (chatContainerEl) {
			chatContainerEl.scrollTop = chatContainerEl.scrollHeight;
		}
	};

	const stopGeneration = () => {
		if (abortController) {
			abortController.abort();
			abortController = null;
			loading = false;
		}
	};

	const sendPrompt = async (customPrompt?: string) => {
		const userText = (customPrompt || prompt).trim();
		if (!userText || loading || !selectedModel) return;

		prompt = '';

		let contextSystemPrompt = `You are an expert AI software engineer coding assistant inside an IDE.
The user is currently viewing/editing the file: "${activeFilePath ?? 'untitled'}"
Here is the current content of the file:
\`\`\`
${activeFileContent || '(file is empty)'}
\`\`\`
Provide concise, correct, production-grade code and explanations. When providing complete code updates, output the code inside markdown code blocks with the appropriate language tag.`;

		const newMessages: Message[] = [
			...messages,
			{ role: 'user', content: userText }
		];

		messages = newMessages;
		loading = true;
		await scrollToBottom();

		const apiMessages = [
			{ role: 'system', content: contextSystemPrompt },
			...newMessages.map((m) => ({ role: m.role, content: m.content }))
		];

		const assistantIndex = messages.length;
		messages = [...messages, { role: 'assistant', content: '' }];

		try {
			const token = localStorage.getItem('token') ?? '';
			const [res, controller] = await chatCompletion(
				token,
				{
					model: selectedModel,
					messages: apiMessages,
					stream: true
				}
			);
			abortController = controller;

			if (res && res.body) {
				const stream = await createOpenAITextStream(res.body, false);
				for await (const update of stream) {
					if (update.value) {
						messages[assistantIndex].content += update.value;
						messages = [...messages];
						await scrollToBottom();
					}
				}
			} else {
				messages[assistantIndex].content = 'Error: Failed to receive response from model.';
			}
		} catch (err: any) {
			if (err?.name !== 'AbortError') {
				console.error('Agent chat error:', err);
				messages[assistantIndex].content = `Error: ${err?.message ?? 'Failed to complete request'}`;
			}
		} finally {
			loading = false;
			abortController = null;
			await scrollToBottom();
		}
	};

	const extractCodeBlocks = (markdown: string): string[] => {
		const matches: string[] = [];
		const regex = /```[\w]*\n([\s\S]*?)```/g;
		let match;
		while ((match = regex.exec(markdown)) !== null) {
			matches.push(match[1].trim());
		}
		return matches;
	};

	const handleApplyCode = (code: string) => {
		if (onApplyCode) {
			onApplyCode(code);
			toast.success($i18n.t('Applied to editor'));
		}
	};

	const clearChat = () => {
		stopGeneration();
		messages = [];
	};
</script>

<div class="flex flex-col h-full bg-white dark:bg-gray-900 border-l border-gray-100 dark:border-gray-850 select-none text-xs">
	<!-- Assistant Header -->
	<div class="flex items-center justify-between px-3 py-2 border-b border-gray-100 dark:border-gray-850 bg-gray-50/50 dark:bg-gray-850/50">
		<div class="flex items-center gap-1.5 min-w-0">
			<svg class="size-4 text-purple-500 shrink-0" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
				<path d="M12 2l3.09 6.26L22 9.27l-5 4.87 1.18 6.88L12 17.77l-6.18 3.25L7 14.14 2 9.27l6.91-1.01L12 2z"/>
			</svg>
			<span class="font-medium text-gray-800 dark:text-gray-200 uppercase tracking-wider text-[11px]">
				{$i18n.t('AI Agent')}
			</span>
		</div>

		<div class="flex items-center gap-1">
			<!-- Model Selector -->
			{#if $models && $models.length > 0}
				<select
					bind:value={selectedModel}
					class="text-[11px] bg-white dark:bg-gray-800 border border-gray-200 dark:border-gray-700 rounded px-1.5 py-0.5 max-w-[120px] truncate text-gray-700 dark:text-gray-300 outline-none"
				>
					{#each $models as m (m.id)}
						<option value={m.id}>{m.name || m.id}</option>
					{/each}
				</select>
			{/if}

			<button
				type="button"
				class="p-1 rounded hover:bg-gray-200 dark:hover:bg-gray-700 text-gray-400 hover:text-gray-600 dark:hover:text-gray-300 transition"
				title={$i18n.t('Clear chat')}
				on:click={clearChat}
			>
				<svg class="size-3.5" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
					<polyline points="3 6 5 6 21 6"/>
					<path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"/>
				</svg>
			</button>
		</div>
	</div>

	<!-- Messages Container -->
	<div
		bind:this={chatContainerEl}
		class="flex-1 overflow-y-auto p-3 space-y-3 select-text"
	>
		{#if messages.length === 0}
			<div class="h-full flex flex-col items-center justify-center text-center p-4 space-y-3 text-gray-400 select-none">
				<div class="size-10 rounded-2xl bg-purple-50 dark:bg-purple-950/40 flex items-center justify-center text-purple-500">
					<svg class="size-5" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
						<path d="M12 2l3.09 6.26L22 9.27l-5 4.87 1.18 6.88L12 17.77l-6.18 3.25L7 14.14 2 9.27l6.91-1.01L12 2z"/>
					</svg>
				</div>
				<div class="space-y-1">
					<p class="font-medium text-xs text-gray-700 dark:text-gray-300">
						{$i18n.t('Ollama & AI Code Agent')}
					</p>
					<p class="text-[11px] text-gray-400">
						{$i18n.t('Ask questions or generate code for the active file')}
					</p>
				</div>

				<!-- Prompt Action Chips -->
				<div class="flex flex-col gap-1.5 w-full max-w-[220px] pt-2">
					<button
						type="button"
						class="px-2.5 py-1.5 rounded-lg bg-gray-100 dark:bg-gray-800 hover:bg-purple-50 dark:hover:bg-purple-950/30 text-gray-600 dark:text-gray-300 hover:text-purple-600 dark:hover:text-purple-400 text-left text-[11px] transition-colors flex items-center gap-1.5"
						on:click={() => sendPrompt('Explain what this code does and its structure')}
					>
						<span>💡</span> <span>{$i18n.t('Explain File')}</span>
					</button>
					<button
						type="button"
						class="px-2.5 py-1.5 rounded-lg bg-gray-100 dark:bg-gray-800 hover:bg-purple-50 dark:hover:bg-purple-950/30 text-gray-600 dark:text-gray-300 hover:text-purple-600 dark:hover:text-purple-400 text-left text-[11px] transition-colors flex items-center gap-1.5"
						on:click={() => sendPrompt('Find any potential bugs or edge cases in this code and provide fixes')}
					>
						<span>🐛</span> <span>{$i18n.t('Find Bugs & Fix')}</span>
					</button>
					<button
						type="button"
						class="px-2.5 py-1.5 rounded-lg bg-gray-100 dark:bg-gray-800 hover:bg-purple-50 dark:hover:bg-purple-950/30 text-gray-600 dark:text-gray-300 hover:text-purple-600 dark:hover:text-purple-400 text-left text-[11px] transition-colors flex items-center gap-1.5"
						on:click={() => sendPrompt('Refactor and optimize this code for better readability and performance')}
					>
						<span>⚡</span> <span>{$i18n.t('Refactor Code')}</span>
					</button>
					<button
						type="button"
						class="px-2.5 py-1.5 rounded-lg bg-gray-100 dark:bg-gray-800 hover:bg-purple-50 dark:hover:bg-purple-950/30 text-gray-600 dark:text-gray-300 hover:text-purple-600 dark:hover:text-purple-400 text-left text-[11px] transition-colors flex items-center gap-1.5"
						on:click={() => sendPrompt('Write comprehensive unit tests for this code')}
					>
						<span>🧪</span> <span>{$i18n.t('Generate Tests')}</span>
					</button>
				</div>
			</div>
		{:else}
			{#each messages as msg, i (i)}
				<div class="space-y-1">
					<div class="flex items-center gap-1 font-semibold text-[10px] uppercase tracking-wide {msg.role === 'user' ? 'text-blue-500' : 'text-purple-500'}">
						{#if msg.role === 'user'}
							<span>{$i18n.t('You')}</span>
						{:else}
							<span>{$i18n.t('Agent')} ({selectedModel})</span>
						{/if}
					</div>

					<div class="text-xs text-gray-800 dark:text-gray-200 leading-relaxed bg-gray-50 dark:bg-gray-850/60 p-2.5 rounded-lg prose dark:prose-invert max-w-none text-[12px]">
						{@html DOMPurify.sanitize(marked.parse(msg.content))}
					</div>

					<!-- If assistant produced code blocks, offer Apply button -->
					{#if msg.role === 'assistant'}
						{@const codeBlocks = extractCodeBlocks(msg.content)}
						{#if codeBlocks.length > 0 && onApplyCode}
							<div class="flex items-center gap-1 pt-0.5">
								<button
									type="button"
									class="inline-flex items-center gap-1 px-2 py-0.5 rounded bg-purple-100 dark:bg-purple-900/30 text-purple-700 dark:text-purple-300 hover:bg-purple-200 dark:hover:bg-purple-900/50 text-[11px] font-medium transition"
									on:click={() => handleApplyCode(codeBlocks[0])}
								>
									<svg class="size-3" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
										<polyline points="20 6 9 17 4 12"/>
									</svg>
									{$i18n.t('Apply Code to Editor')}
								</button>
							</div>
						{/if}
					{/if}
				</div>
			{/each}

			{#if loading && messages[messages.length - 1]?.content === ''}
				<div class="flex items-center gap-2 text-gray-400 text-xs p-2">
					<Spinner className="size-3" />
					<span>{$i18n.t('Thinking...')}</span>
				</div>
			{/if}
		{/if}
	</div>

	<!-- Prompt Input Bar -->
	<div class="p-2 border-t border-gray-100 dark:border-gray-850 bg-white dark:bg-gray-900">
		<form
			class="flex flex-col gap-1.5"
			on:submit|preventDefault={() => sendPrompt()}
		>
			{#if activeFilePath}
				<div class="flex items-center gap-1 text-[10px] text-gray-400 px-1 truncate">
					<span>📎</span> <span class="truncate">{activeFilePath.split('/').pop()}</span>
				</div>
			{/if}

			<div class="relative flex items-center">
				<textarea
					bind:value={prompt}
					rows="2"
					placeholder={$i18n.t('Ask agent or edit code...')}
					class="w-full bg-gray-50 dark:bg-gray-800 border border-gray-200 dark:border-gray-700 rounded-lg px-2.5 py-1.5 text-xs text-gray-800 dark:text-gray-200 resize-none outline-none focus:border-purple-500"
					on:keydown={(e) => {
						if (e.key === 'Enter' && !e.shiftKey) {
							e.preventDefault();
							sendPrompt();
						}
					}}
				/>

				{#if loading}
					<button
						type="button"
						class="absolute right-2 bottom-2 p-1 rounded-md bg-red-500 text-white hover:bg-red-600 transition"
						title={$i18n.t('Stop')}
						on:click={stopGeneration}
					>
						<svg class="size-3" viewBox="0 0 24 24" fill="currentColor">
							<rect x="6" y="6" width="12" height="12" rx="1" />
						</svg>
					</button>
				{:else}
					<button
						type="submit"
						disabled={!prompt.trim()}
						class="absolute right-2 bottom-2 p-1 rounded-md bg-purple-600 text-white hover:bg-purple-700 disabled:opacity-30 disabled:cursor-not-allowed transition"
						title={$i18n.t('Send')}
					>
						<svg class="size-3" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
							<line x1="22" y1="2" x2="11" y2="13"/>
							<polygon points="22 2 15 22 11 13 2 9 22 2"/>
						</svg>
					</button>
				{/if}
			</div>
		</form>
	</div>
</div>
