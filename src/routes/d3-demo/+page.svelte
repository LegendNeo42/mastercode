<script lang="ts">
	import * as d3 from 'd3';
	import { onMount } from 'svelte';

	let svgEl: SVGSVGElement; // Referenz auf das SVG
	let tooltipEl: HTMLDivElement; // Referenz auf das Tooltip-DIV
	let data = [
		{ label: 'Fahrrad', value: 4 },
		{ label: 'Zug', value: 8 },
		{ label: 'Auto', value: 15 },
		{ label: 'Motorrad', value: 16 },
		{ label: 'Bus', value: 23 },
		{ label: 'E-Scooter', value: 42 }
	];
	let xScale!: d3.ScaleBand<string>;
	let gxSel: d3.Selection<SVGGElement, unknown, null, undefined>;
	let barsSel: d3.Selection<SVGRectElement, { label: string; value: number }, SVGGElement, unknown>;
	let sortAsc = true;
	const colorScale = d3
		.scaleOrdinal<string>()
		.domain(data.map((d) => d.label))
		.range(d3.schemeTableau10); // fertiges, gut unterscheidbares Farbschema

	function toggleSort() {
		sortAsc = !sortAsc;
		data.sort((a, b) => (sortAsc ? a.value - b.value : b.value - a.value));
		xScale.domain(data.map((d) => d.label));
		// Balken sanft an neue x-Position schieben
		barsSel
			.transition()
			.duration(600)
			.attr('x', (d) => xScale(d.label)!);
		// x-Achse mitziehen
		gxSel.transition().duration(800).call(d3.axisBottom(xScale).tickSizeOuter(0));
	}

	onMount(() => {
		const width = 500;
		const height = 300;
		const margin = { top: 16, right: 16, bottom: 32, left: 40 };
		const cw = width - margin.left - margin.right; // chart width (innen)
		const ch = height - margin.top - margin.bottom; // chart height (innen)

		xScale = d3
			.scaleBand()
			.domain(data.map((d) => d.label))
			.range([0, cw])
			.padding(0.1);

		const yScale = d3
			.scaleLinear()
			.domain([0, d3.max(data, (d) => d.value)!])
			.range([0, ch]);

		const yScaleAxis = d3
			.scaleLinear()
			.domain([0, d3.max(data, (d) => d.value)!])
			.range([ch, 0]); // ACHTUNG: invertiert für Achse (unten=0, oben=max)

		const svg = d3
			.select(svgEl)
			.attr('width', width)
			.attr('height', height)
			.style('background', '#f8fafc');

		const g = svg.append('g').attr('transform', `translate(${margin.left},${margin.top})`);

		const gy = g.append('g').attr('transform', `translate(0,0)`);
		gy.call(d3.axisLeft(yScaleAxis).ticks(5).tickSizeOuter(0));

		const gx = g.append('g').attr('transform', `translate(0,${ch})`);
		gx.call(d3.axisBottom(xScale).tickSizeOuter(0));

		gxSel = gx; // <- speichern

		const gyGrid = g.append('g').attr('class', 'grid');

		gyGrid
			.call(
				d3
					.axisLeft(yScaleAxis)
					.tickSize(-cw) // Linien über die ganze Breite
					.tickFormat(() => '') // keine Labels
			)
			.selectAll('line')
			.attr('stroke', '#e5e7eb') // Tailwind: slate-200
			.attr('stroke-opacity', 1);

		gyGrid.selectAll('path').remove(); // Achsen-Linie entfernen
		gyGrid.lower(); // Gruppe hinter andere Elemente legen
		type Datum = { label: string; value: number };
		barsSel = g
			.selectAll<SVGRectElement, Datum>('rect')
			.data(data)
			.enter()
			.append('rect')
			.attr('x', (d) => xScale(d.label)!)
			.attr('y', (d) => ch - yScale(d.value))
			.attr('width', xScale.bandwidth())
			.attr('height', (d) => yScale(d.value))
			.attr('fill', (d) => colorScale(d.label)!)
			.on('mouseenter', (event: MouseEvent, d) => {
				d3.select(event.currentTarget as SVGRectElement).attr(
					'fill',
					d3.color(colorScale(d.label)!)!.darker(1).toString()
				);
				tooltipEl.textContent = String(d.value);
				tooltipEl.style.display = 'block';
			})
			.on('mousemove', (event: MouseEvent) => {
				const container = svgEl.parentElement as HTMLElement;
				const rect = container.getBoundingClientRect();
				tooltipEl.style.left = `${event.clientX - rect.left + 12}px`;
				tooltipEl.style.top = `${event.clientY - rect.top + 12}px`;
			})
			.on('mouseleave', (event: MouseEvent, d) => {
				d3.select(event.currentTarget as SVGRectElement).attr('fill', colorScale(d.label)!);
				tooltipEl.style.display = 'none';
			});
		// Legende (oben rechts im Chart)
		const legend = svg
			.append('g')
			.attr('class', 'legend')
			.attr('transform', `translate(${cw - 120}, 0)`);

		const legendItem = legend
			.selectAll('g')
			.data(colorScale.domain())
			.enter()
			.append('g')
			.attr('transform', (_d, i) => `translate(0, ${i * 20})`);

		legendItem
			.append('rect')
			.attr('width', 12)
			.attr('height', 12)
			.attr('rx', 2)
			.attr('fill', (d) => colorScale(d)!);

		legendItem
			.append('text')
			.attr('x', 16)
			.attr('y', 10)
			.attr('font-size', 12)
			.text((d) => d);
	});
</script>

<div class="p-6">
	<h1 class="mb-4 text-xl font-bold">D3 Demo – Balkendiagramm mit Tooltip</h1>

	<!-- Der Wrapper ist relative, damit wir das Tooltip absolut darin positionieren können -->
	<div class="relative inline-block">
		<button class="mb-3 rounded border px-3 py-1.5 text-sm hover:bg-slate-50" onclick={toggleSort}>
			{sortAsc ? 'Nach Wert aufsteigend sortieren' : 'Nach Wert absteigend sortieren'}
		</button>
		<svg bind:this={svgEl}></svg>

		<!-- Tooltip: wird per JS ein-/ausgeblendet & positioniert -->
		<div
			bind:this={tooltipEl}
			class="pointer-events-none absolute z-10 rounded bg-black/80 px-2 py-1 text-xs text-white shadow"
			style="display: none;"
		></div>
	</div>
</div>
