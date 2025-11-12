<script lang="ts">
	import '../app.css';
	let { children } = $props();
</script>

<div class="flex flex-col min-h-screen">
	<main class="main flex flex-col p-1 w-full max-w-[64rem] mx-auto">
		{@render children()}
	</main>
</div>