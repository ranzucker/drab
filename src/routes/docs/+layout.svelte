<script lang="ts">
	export let data;
</script>

<svelte:head>
	<title>drab - {data.title}</title>
</svelte:head>

<h1 class="break-words">{data.title}</h1>

<p>
	<a
		target="_blank"
		href="https://github.com/rossrobino/drab/tree/main/src/lib/components/{data.title}.svelte"
	>
		Source
	</a>
</p>

<div class="card flex justify-center overflow-x-auto border-dashed p-8">
	<div class="w-full">
		<slot />
	</div>
</div>

<div>{@html data.html}</div>
