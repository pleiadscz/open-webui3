<script lang="ts">
	import { onMount, createEventDispatcher } from 'svelte';
	import AddFilesPlaceholder from '../AddFilesPlaceholder.svelte';
	import BottomDrawer from '../common/BottomDrawer.svelte';

	const dispatch = createEventDispatcher();

	export let submitPrompt: Function = () => {};
	export let stopResponse: Function = () => {};
	export let files = [];
	export let fileUploadEnabled = true;
	export let prompt = '';
	export let messages = [];
	export let history = null;
	export let taskIds = null;
	export let generating = false;
	export let placeholder = 'Wyślij wiadomość';

	let drawerOpen = false;
	let inputShellElement: HTMLDivElement;
	let chatTextAreaElement: any; // Can be RichTextInput or textarea
	let filesInputElement;
	let dragged = false;
	let isStopping = false;

	$: lastMessage = history?.currentId ? history?.messages?.[history.currentId] : messages?.at?.(-1);
	$: responsePending =
		generating ||
		(Array.isArray(taskIds) && taskIds.length > 0) ||
		(lastMessage?.role === 'assistant' && lastMessage?.done !== true);

	const uploadFile = (file) => {
		files = [...files, { type: 'doc', name: file.name, upload_status: true }];
	};

	const handleSubmit = () => {
		if (responsePending || isStopping) {
			return;
		}

		if (prompt.trim()) {
			// Call the prop if it exists
			if (submitPrompt) {
				submitPrompt(prompt, null);
			}
			// Dispatch event for components like Chat.svelte and Placeholder.svelte
			dispatch('submit', prompt);
			prompt = '';
		}
	};

	const handleStopResponse = async () => {
		if (isStopping) {
			return;
		}

		isStopping = true;
		try {
			await stopResponse();
		} finally {
			isStopping = false;
		}
	};

	const updateChatInputHeight = () => {
		if (typeof window === 'undefined' || !inputShellElement) {
			return;
		}

		document.documentElement.style.setProperty(
			'--chat-input-height',
			`${Math.ceil(inputShellElement.offsetHeight)}px`
		);
	};

	const updateKeyboardInset = () => {
		if (typeof window === 'undefined') {
			return;
		}

		const visualViewport = window.visualViewport;
		const keyboardInset = visualViewport
			? Math.max(0, window.innerHeight - visualViewport.height - visualViewport.offsetTop)
			: 0;

		document.documentElement.style.setProperty(
			'--chat-keyboard-inset',
			`${Math.round(keyboardInset)}px`
		);
		updateChatInputHeight();
	};

	const handleKeyDown = (e: KeyboardEvent) => {
		if (e.key === 'Enter' && !e.shiftKey && window.innerWidth >= 768) {
			e.preventDefault();
			handleSubmit();
		}
	};

	onMount(() => {
		if (window.innerWidth >= 768) {
			window.setTimeout(() => chatTextAreaElement?.focus(), 0);
		}

		let resizeObserver: ResizeObserver | null = null;

		updateKeyboardInset();
		updateChatInputHeight();
		if (inputShellElement && typeof ResizeObserver !== 'undefined') {
			resizeObserver = new ResizeObserver(updateChatInputHeight);
			resizeObserver.observe(inputShellElement);
		}

		window.visualViewport?.addEventListener('resize', updateKeyboardInset);
		window.visualViewport?.addEventListener('scroll', updateKeyboardInset);
		window.addEventListener('resize', updateKeyboardInset);
		
		const onDrop = (e) => {
			e.preventDefault();
			if (e.dataTransfer?.files) {
				Array.from(e.dataTransfer.files).forEach(uploadFile);
			}
			dragged = false;
		};

		window.addEventListener('dragover', (e) => { e.preventDefault(); dragged = true; });
		window.addEventListener('dragleave', () => { dragged = false; });
		window.addEventListener('drop', onDrop);

		return () => {
			window.visualViewport?.removeEventListener('resize', updateKeyboardInset);
			window.visualViewport?.removeEventListener('scroll', updateKeyboardInset);
			window.removeEventListener('resize', updateKeyboardInset);
			resizeObserver?.disconnect();
			window.removeEventListener('dragover', (e) => e.preventDefault());
			window.removeEventListener('dragleave', () => dragged = false);
			window.removeEventListener('drop', onDrop);
		};
	});

	export const setText = (text: string) => {
		prompt = text;
	};
</script>

{#if dragged}
	<div class="fixed w-full h-full flex z-50 pointer-events-none">
		<div class="absolute w-full h-full backdrop-blur bg-gray-800/40 flex justify-center">
			<div class="m-auto"><AddFilesPlaceholder /></div>
		</div>
	</div>
{/if}

<div class="mobile-input-spacer" aria-hidden="true"></div>

<div class="chat-input-shell w-full" bind:this={inputShellElement}>
	<div class="bg-white dark:bg-gray-900">
		<div class="max-w-3xl px-2.5 mx-auto">
			<div class="pb-2">
				<input bind:this={filesInputElement} type="file" hidden multiple on:change={(e) => {
					if (e.target.files) Array.from(e.target.files).forEach(uploadFile);
				}} />
				
				<form class="flex items-center gap-2 w-full rounded-full px-3 py-2 border border-neutral-200 dark:border-neutral-800 bg-white dark:bg-neutral-900 shadow-[0_0_40px_rgba(0,0,0,0.04)] dark:shadow-none" 
					on:submit|preventDefault={handleSubmit}>
					
					<div class="contents">
						{#if fileUploadEnabled}
							<div class="shrink-0 flex items-center">
								<button class="flex h-9 w-9 shrink-0 items-center justify-center rounded-full text-neutral-700 dark:text-neutral-200 hover:bg-neutral-100 dark:hover:bg-neutral-800 transition-colors" 
									type="button" on:click={() => window.innerWidth >= 768 ? filesInputElement.click() : (drawerOpen = true)}>
									<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 16 16" fill="currentColor" class="w-[1.2rem] h-[1.2rem]">
										<path d="M8.75 3.75a.75.75 0 0 0-1.5 0v3.5h-3.5a.75.75 0 0 0 0 1.5h3.5v3.5a.75.75 0 0 0 1.5 0v-3.5h3.5a.75.75 0 0 0 0-1.5h-3.5v-3.5Z" />
									</svg>
								</button>
							</div>
						{/if}

						<div class="flex-1 flex items-center">
							<textarea id="chat-textarea" bind:this={chatTextAreaElement} 
								class="w-full bg-transparent px-1 py-2 text-[15px] text-neutral-900 dark:text-neutral-100 placeholder:text-neutral-500 dark:placeholder:text-neutral-400 focus:outline-none disabled:opacity-60 resize-none h-[40px] flex items-center"
								{placeholder} bind:value={prompt} rows="1" on:keydown={handleKeyDown} />
						</div>

						<div class="shrink-0 flex items-center space-x-1">
							{#if !responsePending}
								<button class="flex h-9 w-9 shrink-0 items-center justify-center rounded-full bg-black dark:bg-white text-white dark:text-black hover:bg-neutral-800 dark:hover:bg-neutral-200 transition-colors" type="submit">
									<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 16 16" fill="currentColor" class="w-5 h-5">
										<path fill-rule="evenodd" d="M8 14a.75.75 0 0 1-.75-.75V4.56L4.03 7.78a.75.75 0 0 1-1.06-1.06l4.5-4.5a.75.75 0 0 1 1.06 0l4.5 4.5a.75.75 0 0 1-1.06 1.06L8.75 4.56v8.69A.75.75 0 0 1 8 14Z" clip-rule="evenodd" />
									</svg>
								</button>
							{:else}
								<button class="flex h-9 w-9 shrink-0 items-center justify-center rounded-full bg-black dark:bg-white text-white dark:text-black hover:bg-neutral-800 dark:hover:bg-neutral-200 transition-colors disabled:opacity-60" type="button" on:click={handleStopResponse} disabled={isStopping} aria-label="Zatrzymaj generowanie">
									<svg width="24" height="24" viewBox="0 0 24 24" fill="currentColor" class="w-5 h-5"><path d="M6 8C6 6.89543 6.89543 6 8 6H16C17.1046 6 18 6.89543 18 8V16C18 17.1046 17.1046 18 16 18H8C6.89543 18 6 17.1046 6 16V8Z" /></svg>
								</button>
							{/if}
						</div>
					</div>
				</form>
			</div>
		</div>
	</div>
</div>

<BottomDrawer bind:open={drawerOpen}>
	<button class="w-full p-4 text-left" on:click={() => { drawerOpen = false; filesInputElement.click(); }}>Wybierz pliki</button>
</BottomDrawer>


<style>
	.mobile-input-spacer {
		display: none;
	}

	@media (max-width: 767px) {
		.mobile-input-spacer {
			display: block;
			height: calc(var(--chat-input-height, 88px) + env(safe-area-inset-bottom) + 12px);
			flex-shrink: 0;
		}

		.chat-input-shell {
			position: fixed;
			left: 0;
			right: 0;
			bottom: calc(var(--chat-keyboard-inset, 0px) + env(safe-area-inset-bottom) + 12px);
			z-index: 60;
			transition: bottom 120ms ease-out;
		}
	}

	#chat-textarea {
		line-height: 24px;
		overflow-y: auto;
	}
</style>
