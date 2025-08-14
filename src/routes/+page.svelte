<script lang="ts">
	let count = $state(0);
</script>

<div class="min-h-screen grid place-items-center bg-slate-100">
	<div class="max-w-md w-full p-6 rounded-2xl shadow border bg-white">
		<h1 class="text-2xl font-bold tracking-tight mb-2">Svelte + Tailwind ok?</h1>
		<p class="text-slate-600 mb-4">Wenn diese Box gestylt ist, läuft Tailwind.</p>

		<button
			class="px-4 py-2 rounded-lg border text-sm font-medium hover:bg-slate-50 active:scale-[.98] transition"
			onclick={() => (count += 1)}
		>
			Klicks: {count}
		</button>
	</div>
</div>
