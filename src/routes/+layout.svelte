<script>
	import '../app.css';
	import favicon from '$lib/assets/favicon.svg';
	
	let { children } = $props();
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
</svelte:head>

<main class="overflow-x-hidden w-screen h-screen mx-0 my-10 p-0 overscroll-none">
	{@render children?.()}
</main>
