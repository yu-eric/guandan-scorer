<script lang="ts">
	let { children } = $props();
</script>

<svelte:head>
	<title>Guandan Scorer</title>
	<meta name="viewport" content="width=device-width, initial-scale=1.0" />
</svelte:head>

{@render children()}
