<script lang="ts">
	let { children } = $props();
</script>

{@render children?.()}

<style global lang="scss">
	@import 'normalize.css';
	@import '@fontsource/farro';

	* {
		box-sizing: border-box;
	}

	html,
	body {
		font-family: 'Farro', sans-serif;
		font-size: 16px;
		max-width: 100vw;
		width: 100vw;
	}
</style>
