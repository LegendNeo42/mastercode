<script lang="ts">
	import * as d3 from 'd3';
	import { onMount } from 'svelte';
	const series = ['Prof', 'Wiss. Mitarbeiter', 'Wiss. stützend', 'Studierende'] as const;
	type Series = (typeof series)[number];
	type Row = { label: string } & Record<Series, number>;

	let chartGroup: d3.Selection<SVGGElement, unknown, null, undefined>;
	let tooltipDiv: HTMLDivElement;
	let xScaleCategories: d3.ScaleBand<string>;
	// Status, ob Gruppe sichtbar ist
	let visibleGroups = new Set(Array.from(series)); // war vorher evtl. in onMount

	let sortCategoriesAsc = true;
	function toggleSortCategories() {
		sortCategoriesAsc = !sortCategoriesAsc;

		// 1) aktive Gruppen ermitteln
		const active = Array.from(visibleGroups);

		// 2) Summe je Kategorie (nur aktive Gruppen)
		const sums = new Map<string, number>(
			data.map((row) => [row.label, active.reduce((acc, g) => acc + row[g], 0)])
		);

		// 3) neue Domain der Kategorien nach Summe sortieren
		const newDomain = [...xScaleCategories.domain()].sort((a, b) => {
			const sa = sums.get(a) ?? 0;
			const sb = sums.get(b) ?? 0;
			return sortCategoriesAsc ? sa - sb : sb - sa;
		});
		xScaleCategories.domain(newDomain);

		// 4) Kategorien-Gruppen sanft an neue x-Position schieben
		chartGroup
			.selectAll<SVGGElement, any>('g.category')
			.transition()
			.duration(600)
			.ease(d3.easeCubicInOut)
			.attr('transform', (d: any) => `translate(${xScaleCategories(d.label) ?? 0},0)`);

		// 5) X-Achse mit animieren
		chartGroup
			.select<SVGGElement>('.x-axis')
			.transition()
			.duration(600)
			.ease(d3.easeCubicInOut)
			.call(d3.axisBottom(xScaleCategories));
	}

	const data: Row[] = [
		{ label: 'zu Fuß', Prof: 2, 'Wiss. Mitarbeiter': 4, 'Wiss. stützend': 3, Studierende: 5 },
		{ label: 'Fahrrad', Prof: 20, 'Wiss. Mitarbeiter': 28, 'Wiss. stützend': 18, Studierende: 25 },
		{ label: 'E-Bike', Prof: 12, 'Wiss. Mitarbeiter': 15, 'Wiss. stützend': 10, Studierende: 14 },
		{ label: 'Bus', Prof: 18, 'Wiss. Mitarbeiter': 22, 'Wiss. stützend': 12, Studierende: 20 },
		{
			label: 'Zug (Nah)',
			Prof: 350,
			'Wiss. Mitarbeiter': 150,
			'Wiss. stützend': 120,
			Studierende: 170
		},
		{
			label: 'Zug (Fern)',
			Prof: 650,
			'Wiss. Mitarbeiter': 540,
			'Wiss. stützend': 80,
			Studierende: 280
		},
		{
			label: 'Auto (Fahrer)',
			Prof: 60,
			'Wiss. Mitarbeiter': 75,
			'Wiss. stützend': 90,
			Studierende: 70
		},
		{
			label: 'Auto (Beif.)',
			Prof: 15,
			'Wiss. Mitarbeiter': 160,
			'Wiss. stützend': 55,
			Studierende: 65
		},
		{ label: 'Motorrad', Prof: 35, 'Wiss. Mitarbeiter': 45, 'Wiss. stützend': 55, Studierende: 40 }
	];
	const originalCategoryOrder = data.map((d) => d.label);
	// Gesamtgröße des SVG
	const svgWidth = 800;
	const svgHeight = 500;

	// Außenränder für Achsenbeschriftungen & Legende
	const margin = { top: 40, right: 150, bottom: 50, left: 60 };

	// Innenfläche, auf der die Balken & Achsen liegen
	const chartWidth = svgWidth - margin.left - margin.right;
	const chartHeight = svgHeight - margin.top - margin.bottom;

	let svgElement: SVGSVGElement;

	onMount(() => {
		// Erstelle Haupt-SVG
		const svg = d3.select(svgElement).attr('width', svgWidth).attr('height', svgHeight);

		// Erstelle eine verschobene Gruppe für den Chart-Bereich
		chartGroup = svg
			.append('g')
			.attr('transform', `translate(${margin.left},${margin.top})`)
			.attr('class', 'chart-area');
		chartGroup
			.append('rect')
			.attr('width', chartWidth)
			.attr('height', chartHeight)
			.attr('fill', '#f8fafc') // heller Hintergrund
			.attr('stroke', '#e5e7eb') // zarter Rand
			.attr('rx', 6);

		// 1) Verkehrsmittel (große Gruppen auf der X-Achse)
		xScaleCategories = d3
			.scaleBand<string>()
			.domain(data.map((d) => d.label)) // ["zu Fuß","Fahrrad",...]
			.range([0, chartWidth])
			.paddingInner(0.2) // Abstand zwischen den Kategorien
			.paddingOuter(0.05);

		// 2) Status-Gruppen innerhalb einer Kategorie (Teilbalken)
		const xScaleGroups = d3
			.scaleBand<string>()
			.domain(Array.from(series)) // ["Prof","Wiss. Mitarbeiter",...]
			.range([0, xScaleCategories.bandwidth()])
			.padding(0.1); // Abstand zwischen Teilbalken

		// 3) Y-Skala (Werte → Höhe)
		const yMax = d3.max(data, (row) => d3.max(Array.from(series), (s) => row[s]))!;
		const yScale = d3
			.scaleLinear()
			.domain([0, yMax]) // 0 bis Maximalwert
			.range([chartHeight, 0]); // unten→oben (für Achsen korrekt)

		// Y-Achse links (mit Format)
		const fmt = d3.format(',.0f'); // 1 234
		chartGroup
			.append('g')
			.call(
				d3
					.axisLeft(yScale)
					.ticks(6)
					.tickFormat((d) => `${fmt(d as number)} km`)
			)
			.attr('class', 'y-axis');
		// Y-Achsen-Beschriftung
		chartGroup
			.append('text')
			.attr('class', 'y-axis-label')
			.attr('x', -margin.left + 10) // etwas links vom Diagramm
			.attr('y', -10) // oberhalb der Y-Achse
			.attr('text-anchor', 'start')
			.attr('font-size', 14)
			.attr('font-weight', 'bold')
			.text('Distanz (km)');

		// X-Achse unten
		chartGroup
			.append('g')
			.attr('transform', `translate(0,${chartHeight})`)
			.call(d3.axisBottom(xScaleCategories))
			.attr('class', 'x-axis');

		// X-Labels rotieren für bessere Lesbarkeit
		chartGroup
			.selectAll('.x-axis text')
			.attr('text-anchor', 'end')
			.attr('transform', 'rotate(-25)')
			.attr('dx', '-0.6em')
			.attr('dy', '0.15em');

		// X-Achsen-Beschriftung
		chartGroup
			.append('text')
			.attr('class', 'x-axis-label')
			.attr('x', chartWidth / 2)
			.attr('y', chartHeight + margin.bottom) // etwas unter der Achse
			.attr('text-anchor', 'middle')
			.attr('font-size', 14)
			.attr('font-weight', 'bold')
			.text('Verkehrsmittel');

		// Diagramm-Untertitel im Chart
		chartGroup
			.append('text')
			.attr('class', 'chart-subtitle')
			.attr('x', chartWidth / 2)
			.attr('y', -margin.top / 2) // oberhalb des Diagramms, aber innerhalb des SVG
			.attr('text-anchor', 'middle')
			.attr('font-size', 13)
			.attr('fill', '#374151') // Tailwind slate-700
			.text('Distanz pro Verkehrsmittel und Statusgruppe');

		// Farbskala pro Status-Gruppe
		const colorScale = d3
			.scaleOrdinal<string, string>()
			.domain(Array.from(series))
			.range(d3.schemeTableau10);
		// Eine <g>-Gruppe je Verkehrsmittel (Kategorie)
		const categoryGroups = chartGroup
			.selectAll('g.category')
			.data(data)
			.enter()
			.append('g')
			.attr('class', 'category')
			.attr('transform', (d) => `translate(${xScaleCategories(d.label) ?? 0},0)`);

		// horizontale Gridlines hinter den Balken
		const yGrid = chartGroup.append('g').attr('class', 'y-grid');
		yGrid
			.call(
				d3
					.axisLeft(yScale)
					.ticks(6) // Anzahl Linien (anpassen, z. B. 8)
					.tickSize(-chartWidth)
					.tickFormat(() => '') // keine Labels
			)
			.selectAll('line')
			.attr('stroke', '#e5e7eb') // hellgrau
			.attr('stroke-opacity', 1);
		yGrid.selectAll('path').remove(); // Achsenlinie entfernen
		// Balken über das Grid heben
		chartGroup.selectAll<SVGGElement, unknown>('g.category').raise();

		// In jeder Kategorie: ein <rect> je Status-Gruppe
		categoryGroups
			.selectAll('rect.bar')
			.data((d) => Array.from(series, (s) => ({ group: s, value: d[s] })))
			.enter()
			.append('rect')
			.attr('class', 'bar')
			.attr('data-group', (d) => d.group) // Marker: gehört zu welcher Status-Gruppe
			.attr('x', (d) => xScaleGroups(d.group)!)
			.attr('y', (d) => yScale(d.value))
			.attr('width', xScaleGroups.bandwidth())
			.attr('height', (d) => chartHeight - yScale(d.value))
			.attr('fill', (d) => colorScale(d.group))
			.attr('opacity', 1)

			.on('mouseenter', function (event, d) {
				d3.select(this).attr('opacity', 0.8);
				tooltipDiv = document.getElementById('tooltip') as HTMLDivElement;
				tooltipDiv.style.display = 'block';
				tooltipDiv.textContent = `${d.group} – ${d.value} km`;
				// gesamte Kategorie hervorheben
				const catGroup = (this as SVGRectElement).parentNode as SVGGElement;
				const catData = d3.select<SVGGElement, Row>(catGroup).datum();
				chartGroup
					.selectAll<SVGGElement, Row>('g.category')
					.transition()
					.duration(150)
					.style('opacity', (g) => (g.label === catData.label ? 1 : 0.3));
			})
			.on('mousemove', function (event) {
				tooltipDiv.style.left = event.pageX + 12 + 'px';
				tooltipDiv.style.top = event.pageY + 12 + 'px';
			})
			.on('mouseleave', function () {
				d3.select(this).attr('opacity', 1);
				tooltipDiv.style.display = 'none';
				// Hervorhebung zurücksetzen
				chartGroup.selectAll('g.category').transition().duration(50).style('opacity', 1);
			});
		categoryGroups
			.selectAll('text.bar-label')
			.data((d) => Array.from(series, (s) => ({ group: s, value: d[s] })))
			.enter()
			.append('text')
			.attr('class', 'bar-label')
			.attr('x', (d) => (xScaleGroups(d.group) ?? 0) + xScaleGroups.bandwidth() / 2)
			.attr('y', (d) => yScale(d.value) - 4) // leicht oberhalb des Balkens
			.attr('data-group', (d) => d.group) // Gruppe merken – wie bei den Balken
			.attr('opacity', 1) // definierter Startzustand
			.attr('text-anchor', 'middle')
			.attr('font-size', '10px')
			.attr('fill', '#374151')
			.text((d) => d.value);

		// Legende rechts neben dem Chart
		const legendGroup = svg
			.append('g')
			.attr('class', 'legend')
			.attr('transform', `translate(${margin.left + chartWidth + 20}, ${margin.top})`);

		const legendItem = legendGroup
			.selectAll('g')
			.data(Array.from(series))
			.enter()
			.append('g')
			.attr('transform', (_d, i) => `translate(0, ${i * 20})`)
			.style('cursor', 'pointer')
			.on('click', function (_event, groupName) {
				if (visibleGroups.has(groupName)) {
					visibleGroups.delete(groupName);
				} else {
					visibleGroups.add(groupName);
				}
				updateBars();

				// Legendenfarbe anpassen
				d3.select(this)
					.select('rect')
					.attr('fill', visibleGroups.has(groupName) ? colorScale(groupName) : '#ccc');
			});

		// Farbfelder
		legendItem
			.append('rect')
			.attr('width', 12)
			.attr('height', 12)
			.attr('rx', 2)
			.attr('fill', (d) => colorScale(d));

		// Textlabels
		legendItem
			.append('text')
			.attr('x', 18)
			.attr('y', 10)
			.attr('font-size', 12)
			.text((d) => d);

		// ENDE OnMount
	});

	function updateBars() {
		const activeGroups = Array.from(visibleGroups);

		// Neue xScaleGroups mit nur aktiven Gruppen
		const xScaleGroupsUpdated = d3
			.scaleBand<string>()
			.domain(activeGroups)
			.range([0, xScaleCategories.bandwidth()])
			.padding(0.1);

		// Update alle Balken
		chartGroup
			.selectAll<SVGRectElement, any>('rect.bar')
			.transition()
			.duration(500)
			.attr('x', (d) =>
				activeGroups.includes(d.group)
					? xScaleGroupsUpdated(d.group)!
					: xScaleCategories.bandwidth() / 2
			)
			.attr('width', (d) => (activeGroups.includes(d.group) ? xScaleGroupsUpdated.bandwidth() : 0))
			.attr('opacity', (d) => (activeGroups.includes(d.group) ? 1 : 0));

		chartGroup
			.selectAll<SVGTextElement, { group: string; value: number }>('text.bar-label')
			.transition()
			.duration(500)
			.ease(d3.easeCubicInOut)
			.attr('x', (d) =>
				activeGroups.includes(d.group)
					? xScaleGroupsUpdated(d.group)! + xScaleGroupsUpdated.bandwidth() / 2
					: xScaleCategories.bandwidth() / 2
			) // egaler Platzhalter, wenn ausgeblendet
			.attr('opacity', (d) => (activeGroups.includes(d.group) ? 1 : 0));
	}

	function resetCategoryOrder() {
		// X-Domain auf Ursprungsreihenfolge setzen
		xScaleCategories.domain(originalCategoryOrder);

		// Kategorien sanft zurückschieben
		chartGroup
			.selectAll<SVGGElement, any>('g.category')
			.transition()
			.duration(600)
			.ease(d3.easeCubicInOut)
			.attr('transform', (d: any) => `translate(${xScaleCategories(d.label) ?? 0},0)`);

		// X-Achse mit animieren
		chartGroup
			.select<SVGGElement>('.x-axis')
			.transition()
			.duration(600)
			.ease(d3.easeCubicInOut)
			.call(d3.axisBottom(xScaleCategories));
	}
</script>

<div class="p-6">
	<h1 class="mb-4 text-xl font-bold">Mobilitätsdaten – Universität Regensburg</h1>

	<div class="mb-4 flex gap-2">
		<button
			class="rounded border px-3 py-1.5 text-sm hover:bg-slate-50"
			onclick={resetCategoryOrder}
		>
			Nach Verkehrsmittel sortieren
		</button>
		<button
			class="rounded border px-3 py-1.5 text-sm hover:bg-slate-50"
			onclick={toggleSortCategories}
		>
			{sortCategoriesAsc ? 'Nach Summe absteigend sortieren' : 'Nach Summe aufsteigend sortieren'}
		</button>
	</div>

	<svg bind:this={svgElement}></svg>
	<div
		id="tooltip"
		class="pointer-events-none absolute z-10 rounded bg-black/80 px-2 py-1 text-xs text-white shadow"
		style="display: none;"
	></div>
</div>
