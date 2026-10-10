<script lang="ts">
	import DOMPurify from 'dompurify';

	import { onDestroy, onMount } from 'svelte';

	import tippy, {
		type Instance as TippyInstance,
		type Placement as TippyPlacement,
		type Props as TippyProps
	} from 'tippy.js';

	export let elementId = '';

	export let as = 'div';
	export let className = 'flex';

	export let placement: TippyPlacement = 'top';
	export let content = `I'm a tooltip!`;
	export let touch: TippyProps['touch'] | undefined = undefined;
	export let theme = '';
	export let offset: TippyProps['offset'] = [0, 4];
	export let allowHTML = true;
	export let tippyOptions: Partial<TippyProps> = {};
	export let interactive = false;

	export let onClick = () => {};

	let tooltipElement: HTMLElement | null = null;
	let tooltipInstance: TippyInstance | null = null;

	// tippy's touch behavior shows the tooltip on touchstart and consumes that first
	// tap, so a tooltipped button needs two taps on a touch device (e.g. the sidebar
	// toggle). When no `touch` prop is passed, disable that behavior for interactive
	// controls (button, link, input, ...) so the first tap reaches the control;
	// non-interactive targets keep tippy's default, and any explicit value wins.
	const controlSelector =
		'button, [role="button"], a[href], input, select, textarea, [contenteditable="true"], [contenteditable=""]';

	const isInteractiveTarget = (el: HTMLElement | null) => {
		const probeRoot = el?.firstElementChild ?? el;
		return !!(probeRoot && (typeof probeRoot.matches !== 'function' || probeRoot.matches(controlSelector)));
	};

	// On some iOS standalone-PWA cold opens, WebKit completes the tap's pointer
	// gesture (pointerdown + pointerup) but never synthesizes the click, so the
	// control's own handler never runs and the user has to tap twice. Recover by
	// dispatching a synthetic click for trusted short touch pointerups. A trusted
	// click arriving first supersedes the fallback; a trusted click arriving after
	// the fallback fired is that fallback's duplicate and is dropped, so a control
	// can never fire twice. Only real controls are recovered — text fields are
	// excluded so this can never re-focus an input (and re-open the keyboard).
	const recoverySelector = 'button, [role="button"], a[href]';
	const isTouchDevice = 'ontouchstart' in window || navigator.maxTouchPoints > 0;

	let recoveryElement: HTMLElement | null = null;
	let pointerDownAt = 0;
	let lastTrustedClickAt = 0;
	let fallbackFiredAt = 0;

	const onRecoveryPointerDown = (e: PointerEvent) => {
		if (e.isTrusted) {
			pointerDownAt = Date.now();
		}
	};

	const onRecoveryPointerUp = (e: PointerEvent) => {
		if (!isTouchDevice || !e.isTrusted) {
			return;
		}
		if (e.pointerType && e.pointerType !== 'touch') {
			return;
		}

		const downAt = pointerDownAt;
		pointerDownAt = 0;

		const heldMs = downAt ? Date.now() - downAt : -1;
		if (heldMs < 0 || heldMs > 500) {
			// long-press or unknown gesture: leave native behavior alone
			return;
		}

		const upAt = Date.now();
		window.setTimeout(() => {
			if (lastTrustedClickAt >= upAt - 10) {
				// WebKit dispatched the click after all
				return;
			}
			fallbackFiredAt = Date.now();
			recoveryElement?.click();
		}, 60);
	};

	const onRecoveryClick = (e: MouseEvent) => {
		if (!e.isTrusted) {
			// our own fallback click: let it run
			return;
		}
		lastTrustedClickAt = Date.now();
		if (fallbackFiredAt && Date.now() - fallbackFiredAt < 1000) {
			// late trusted click duplicating the fallback: drop it
			e.preventDefault();
			e.stopPropagation();
		}
	};

	const detachRecovery = () => {
		if (!recoveryElement) {
			return;
		}
		recoveryElement.removeEventListener('pointerdown', onRecoveryPointerDown);
		recoveryElement.removeEventListener('pointerup', onRecoveryPointerUp);
		recoveryElement.removeEventListener('click', onRecoveryClick, true);
		recoveryElement = null;
	};

	const attachRecovery = () => {
		const probeRoot = tooltipElement?.firstElementChild;
		const next =
			probeRoot && typeof probeRoot.matches === 'function' && probeRoot.matches(recoverySelector)
				? probeRoot
				: null;
		if (next === recoveryElement) {
			return;
		}
		detachRecovery();
		if (next) {
			recoveryElement = next;
			next.addEventListener('pointerdown', onRecoveryPointerDown);
			next.addEventListener('pointerup', onRecoveryPointerUp);
			next.addEventListener('click', onRecoveryClick, true);
		}
	};

	function destroyInstance() {
		if (tooltipInstance) {
			tooltipInstance.destroy();
			tooltipInstance = null;
		}
	}

	$: if (tooltipElement && (content || elementId)) {
		attachRecovery();

		let tooltipContent: string | Element | DocumentFragment | null = null;

		if (elementId) {
			tooltipContent = document.getElementById(elementId);
		} else {
			tooltipContent = DOMPurify.sanitize(content);
		}

		// After the element changes, the old instance must be destroyed, otherwise the detached tippy floating DOM will be left behind
		if (tooltipInstance && tooltipInstance.reference !== tooltipElement) {
			destroyInstance();
		}

		if (tooltipInstance) {
			tooltipInstance.setContent(tooltipContent ?? '');
		} else {
			if (content) {
				tooltipInstance = tippy(tooltipElement, {
					content: tooltipContent ?? '',
					placement,
					allowHTML,
					touch: touch === undefined ? !isInteractiveTarget(tooltipElement) : touch,
					...(theme !== '' ? { theme } : { theme: 'dark' }),
					arrow: false,
					offset,
					...(interactive ? { interactive: true } : {}),
					...tippyOptions
				});
			}
		}
	} else if (tooltipInstance && content === '') {
		destroyInstance();
	}

	onMount(() => {
		attachRecovery();
	});

	onDestroy(() => {
		detachRecovery();
		destroyInstance();
	});
</script>

<!-- svelte-ignore a11y-no-static-element-interactions -->
<svelte:element this={as} bind:this={tooltipElement} class={className} on:click={onClick}>
	<slot />
</svelte:element>

<slot name="tooltip"></slot>
