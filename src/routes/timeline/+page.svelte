<script lang="ts">
	import { onMount } from 'svelte';
	import * as d3 from 'd3';
	import eventsData from '$lib/data/timeline-events.json';

	type TimelineEvent = {
		year: number;
		title: string;
		description: string;
		significance: number;
		category: string;
	};

	const events: TimelineEvent[] = eventsData as TimelineEvent[];
	const categories = [...new Set(events.map((e) => e.category))].sort();

	const categoryColors: Record<string, string> = {
		invention: '#22d3ee',
		crisis: '#f87171',
		policy: '#c084fc',
		institution: '#facc15',
		standard: '#4ade80',
		debasement: '#fb923c'
	};

	const categoryLabels: Record<string, string> = {
		invention: 'Inventions',
		crisis: 'Crises',
		policy: 'Policy',
		institution: 'Institutions',
		standard: 'Standards',
		debasement: 'Debasement'
	};

	let container: HTMLDivElement;
	let canvas: HTMLCanvasElement;
	let selectedEvent: TimelineEvent | null = $state(null);
	let activeCategories: Set<string> = $state(new Set(categories));
	let hoveredEvent: TimelineEvent | null = $state(null);
	let currentZoom: d3.ZoomTransform = $state(d3.zoomIdentity);
	let tooltipX: number = $state(0);
	let tooltipY: number = $state(0);

	function formatYear(year: number): string {
		if (year < 0) return `${Math.abs(year)} BC`;
		return `${year} AD`;
	}

	function toggleCategory(cat: string) {
		const next = new Set(activeCategories);
		if (next.has(cat)) {
			if (next.size > 1) next.delete(cat);
		} else {
			next.add(cat);
		}
		activeCategories = next;
		draw();
	}

	function getVisibleEvents(): TimelineEvent[] {
		return events.filter((e) => activeCategories.has(e.category));
	}

	function getMinSignificance(k: number): number {
		if (k < 0.5) return 5;
		if (k < 1) return 4;
		if (k < 2) return 3;
		if (k < 5) return 2;
		return 1;
	}

	let width = 0;
	let height = 0;
	let xScale: d3.ScaleLinear<number, number>;
	let zoomBehavior: d3.ZoomBehavior<HTMLCanvasElement, unknown>;
	let dpr = 1;

	function draw() {
		if (!canvas) return;
		const ctx = canvas.getContext('2d');
		if (!ctx) return;

		ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
		ctx.clearRect(0, 0, width, height);

		const transform = currentZoom;
		const newX = transform.rescaleX(xScale);
		const minSig = getMinSignificance(transform.k);
		const visible = getVisibleEvents().filter((e) => e.significance >= minSig);

		const midY = height / 2;

		// Draw timeline axis
		ctx.strokeStyle = 'rgba(255,255,255,0.08)';
		ctx.lineWidth = 1;
		ctx.beginPath();
		ctx.moveTo(0, midY);
		ctx.lineTo(width, midY);
		ctx.stroke();

		// Draw tick marks and year labels
		const domain = newX.domain();
		const range = domain[1] - domain[0];
		let tickInterval: number;
		if (range > 3000) tickInterval = 1000;
		else if (range > 1000) tickInterval = 500;
		else if (range > 500) tickInterval = 100;
		else if (range > 100) tickInterval = 50;
		else if (range > 50) tickInterval = 10;
		else tickInterval = 5;

		const startTick = Math.ceil(domain[0] / tickInterval) * tickInterval;
		ctx.fillStyle = 'rgba(255,255,255,0.3)';
		ctx.font = '11px Inter, system-ui, sans-serif';
		ctx.textAlign = 'center';

		for (let tick = startTick; tick <= domain[1]; tick += tickInterval) {
			const tx = newX(tick);
			if (tx < -50 || tx > width + 50) continue;

			ctx.strokeStyle = 'rgba(255,255,255,0.05)';
			ctx.beginPath();
			ctx.moveTo(tx, midY - 8);
			ctx.lineTo(tx, midY + 8);
			ctx.stroke();

			const label = tick < 0 ? `${Math.abs(tick)} BC` : `${tick}`;
			ctx.fillText(label, tx, midY + 24);
		}

		// Scatter events vertically by significance and add some deterministic jitter
		function getEventY(e: TimelineEvent): number {
			const hash = e.year * 7 + e.title.charCodeAt(0) * 13;
			const jitter = ((hash % 100) / 100 - 0.5) * 0.6;
			const sigOffset = (5 - e.significance) / 5;
			const spread = Math.min(height * 0.35, 200);
			return midY + (jitter + sigOffset * 0.2) * spread;
		}

		function getDotRadius(e: TimelineEvent): number {
			const base = 3 + e.significance * 3.5;
			const scale = Math.max(0.5, Math.min(2, transform.k * 0.5));
			return base * scale;
		}

		// Draw dots
		for (const e of visible) {
			const ex = newX(e.year);
			if (ex < -50 || ex > width + 50) continue;

			const ey = getEventY(e);
			const r = getDotRadius(e);
			const color = categoryColors[e.category] || '#ffffff';
			const isHovered = hoveredEvent === e;
			const isSelected = selectedEvent === e;

			// Glow
			if (e.significance >= 4 || isHovered || isSelected) {
				const glowR = r * (isHovered || isSelected ? 3 : 2);
				const gradient = ctx.createRadialGradient(ex, ey, 0, ex, ey, glowR);
				const alpha = isHovered || isSelected ? 0.3 : 0.15;
				gradient.addColorStop(0, color.replace(')', `, ${alpha})`).replace('rgb', 'rgba'));
				gradient.addColorStop(1, 'transparent');
				ctx.fillStyle = gradient;
				ctx.beginPath();
				ctx.arc(ex, ey, glowR, 0, Math.PI * 2);
				ctx.fill();
			}

			// Dot
			ctx.fillStyle = isHovered || isSelected ? '#ffffff' : color;
			ctx.globalAlpha = isHovered || isSelected ? 1 : 0.6 + e.significance * 0.08;
			ctx.beginPath();
			ctx.arc(ex, ey, r, 0, Math.PI * 2);
			ctx.fill();
			ctx.globalAlpha = 1;

			// Label for significant events when zoomed in enough
			if ((e.significance >= 4 && transform.k > 1) || (e.significance >= 3 && transform.k > 3) || isHovered || isSelected) {
				ctx.fillStyle = isHovered || isSelected ? '#ffffff' : 'rgba(255,255,255,0.7)';
				ctx.font = `${isHovered || isSelected ? '13' : '11'}px Inter, system-ui, sans-serif`;
				ctx.textAlign = 'center';
				ctx.fillText(e.title, ex, ey - r - 8);
				ctx.font = '10px Inter, system-ui, sans-serif';
				ctx.fillStyle = 'rgba(255,255,255,0.4)';
				ctx.fillText(formatYear(e.year), ex, ey - r - 22);
			}
		}
	}

	function handleCanvasClick(event: MouseEvent) {
		const rect = canvas.getBoundingClientRect();
		const mx = event.clientX - rect.left;
		const my = event.clientY - rect.top;
		const newX = currentZoom.rescaleX(xScale);
		const minSig = getMinSignificance(currentZoom.k);
		const visible = getVisibleEvents().filter((e) => e.significance >= minSig);

		const midY = height / 2;
		let closest: TimelineEvent | null = null;
		let closestDist = Infinity;

		for (const e of visible) {
			const ex = newX(e.year);
			const hash = e.year * 7 + e.title.charCodeAt(0) * 13;
			const jitter = ((hash % 100) / 100 - 0.5) * 0.6;
			const sigOffset = (5 - e.significance) / 5;
			const spread = Math.min(height * 0.35, 200);
			const ey = midY + (jitter + sigOffset * 0.2) * spread;
			const r = (3 + e.significance * 3.5) * Math.max(0.5, Math.min(2, currentZoom.k * 0.5));

			const dist = Math.sqrt((mx - ex) ** 2 + (my - ey) ** 2);
			if (dist < r + 10 && dist < closestDist) {
				closest = e;
				closestDist = dist;
			}
		}

		selectedEvent = closest;
		draw();
	}

	function handleCanvasMove(event: MouseEvent) {
		const rect = canvas.getBoundingClientRect();
		const mx = event.clientX - rect.left;
		const my = event.clientY - rect.top;
		const newX = currentZoom.rescaleX(xScale);
		const minSig = getMinSignificance(currentZoom.k);
		const visible = getVisibleEvents().filter((e) => e.significance >= minSig);

		const midY = height / 2;
		let closest: TimelineEvent | null = null;
		let closestDist = Infinity;

		for (const e of visible) {
			const ex = newX(e.year);
			const hash = e.year * 7 + e.title.charCodeAt(0) * 13;
			const jitter = ((hash % 100) / 100 - 0.5) * 0.6;
			const sigOffset = (5 - e.significance) / 5;
			const spread = Math.min(height * 0.35, 200);
			const ey = midY + (jitter + sigOffset * 0.2) * spread;
			const r = (3 + e.significance * 3.5) * Math.max(0.5, Math.min(2, currentZoom.k * 0.5));

			const dist = Math.sqrt((mx - ex) ** 2 + (my - ey) ** 2);
			if (dist < r + 10 && dist < closestDist) {
				closest = e;
				closestDist = dist;
			}
		}

		const prev = hoveredEvent;
		hoveredEvent = closest;
		canvas.style.cursor = closest ? 'pointer' : 'grab';
		if (closest) {
			tooltipX = event.clientX;
			tooltipY = event.clientY;
		}
		if (prev !== closest) draw();
	}

	onMount(() => {
		dpr = window.devicePixelRatio || 1;

		function resize() {
			width = container.clientWidth;
			height = container.clientHeight;
			canvas.width = width * dpr;
			canvas.height = height * dpr;
			canvas.style.width = `${width}px`;
			canvas.style.height = `${height}px`;

			xScale = d3.scaleLinear().domain([-3200, 2030]).range([60, width - 60]);

			draw();
		}

		resize();
		window.addEventListener('resize', resize);

		zoomBehavior = d3
			.zoom<HTMLCanvasElement, unknown>()
			.scaleExtent([0.3, 100])
			.on('zoom', (event) => {
				currentZoom = event.transform;
				draw();
			});

		d3.select(canvas).call(zoomBehavior);

		canvas.addEventListener('click', handleCanvasClick);
		canvas.addEventListener('mousemove', handleCanvasMove);

		return () => {
			window.removeEventListener('resize', resize);
			canvas.removeEventListener('click', handleCanvasClick);
			canvas.removeEventListener('mousemove', handleCanvasMove);
		};
	});
</script>

<svelte:head>
	<title>Timeline of Money — MoneyedBot</title>
	<meta name="description" content="An interactive timeline of the history of money, from ancient Mesopotamia to modern central banking." />
</svelte:head>

<div class="timeline-page">
	<!-- Header -->
	<div class="header">
		<h1>The History of Money</h1>
		<p class="subtitle">Scroll to explore. Zoom to discover. Click for details.</p>
	</div>

	<!-- Category Filters -->
	<div class="filters">
		{#each categories as cat}
			<button
				class="filter-btn"
				class:active={activeCategories.has(cat)}
				style="--cat-color: {categoryColors[cat]}"
				onclick={() => toggleCategory(cat)}
			>
				<span class="dot" style="background: {categoryColors[cat]}"></span>
				{categoryLabels[cat] || cat}
			</button>
		{/each}
	</div>

	<!-- Timeline Canvas -->
	<div class="canvas-container" bind:this={container}>
		<canvas bind:this={canvas}></canvas>
	</div>

	<!-- Zoom hint -->
	<div class="zoom-hint">
		<span>Scroll to zoom · Drag to pan</span>
	</div>

	<!-- Hover Tooltip -->
	{#if hoveredEvent && !selectedEvent}
		<div
			class="tooltip"
			style="left: {tooltipX}px; top: {tooltipY}px"
		>
			<div class="tooltip-category" style="color: {categoryColors[hoveredEvent.category]}">
				{categoryLabels[hoveredEvent.category] || hoveredEvent.category} · {formatYear(hoveredEvent.year)}
			</div>
			<div class="tooltip-title">{hoveredEvent.title}</div>
			<div class="tooltip-hint">Click for details</div>
		</div>
	{/if}

	<!-- Detail Panel -->
	{#if selectedEvent}
		<div class="detail-panel" role="dialog">
			<button class="close-btn" onclick={() => { selectedEvent = null; draw(); }}>✕</button>
			<div class="detail-category" style="color: {categoryColors[selectedEvent.category]}">
				{categoryLabels[selectedEvent.category] || selectedEvent.category}
			</div>
			<div class="detail-year">{formatYear(selectedEvent.year)}</div>
			<h2 class="detail-title">{selectedEvent.title}</h2>
			<p class="detail-desc">{selectedEvent.description}</p>
			<div class="detail-significance">
				{#each Array(5) as _, i}
					<span class="sig-dot" class:filled={i < selectedEvent.significance}></span>
				{/each}
				<span class="sig-label">Significance</span>
			</div>
		</div>
	{/if}
</div>

<style>
	.timeline-page {
		position: fixed;
		inset: 0;
		background: #050507;
		display: flex;
		flex-direction: column;
		overflow: hidden;
		font-family: 'Inter', system-ui, sans-serif;
	}

	.header {
		text-align: center;
		padding: 20px 20px 8px;
		z-index: 10;
	}

	.header h1 {
		font-size: 1.4rem;
		font-weight: 600;
		color: #e5e5e5;
		margin: 0;
		letter-spacing: -0.02em;
	}

	.subtitle {
		font-size: 0.8rem;
		color: rgba(255, 255, 255, 0.35);
		margin: 4px 0 0;
	}

	.filters {
		display: flex;
		justify-content: center;
		gap: 6px;
		padding: 10px 20px;
		flex-wrap: wrap;
		z-index: 10;
	}

	.filter-btn {
		display: flex;
		align-items: center;
		gap: 5px;
		padding: 4px 10px;
		border: 1px solid rgba(255, 255, 255, 0.1);
		border-radius: 20px;
		background: rgba(255, 255, 255, 0.03);
		color: rgba(255, 255, 255, 0.5);
		font-size: 0.7rem;
		cursor: pointer;
		transition: all 0.2s;
	}

	.filter-btn:hover {
		border-color: var(--cat-color);
		color: rgba(255, 255, 255, 0.8);
	}

	.filter-btn.active {
		border-color: var(--cat-color);
		background: rgba(255, 255, 255, 0.06);
		color: #e5e5e5;
	}

	.filter-btn .dot {
		width: 7px;
		height: 7px;
		border-radius: 50%;
		opacity: 0.7;
	}

	.filter-btn.active .dot {
		opacity: 1;
	}

	.canvas-container {
		flex: 1;
		position: relative;
		min-height: 0;
	}

	canvas {
		position: absolute;
		inset: 0;
	}

	.zoom-hint {
		text-align: center;
		padding: 8px;
		z-index: 10;
	}

	.zoom-hint span {
		font-size: 0.7rem;
		color: rgba(255, 255, 255, 0.2);
	}

	.tooltip {
		position: fixed;
		transform: translate(-50%, -100%);
		margin-top: -16px;
		padding: 10px 14px;
		background: rgba(15, 15, 20, 0.95);
		border: 1px solid rgba(255, 255, 255, 0.12);
		border-radius: 8px;
		z-index: 90;
		pointer-events: none;
		backdrop-filter: blur(12px);
		max-width: 260px;
		animation: tooltipIn 0.12s ease-out;
	}

	@keyframes tooltipIn {
		from {
			opacity: 0;
			transform: translate(-50%, -100%) translateY(4px);
		}
		to {
			opacity: 1;
			transform: translate(-50%, -100%) translateY(0);
		}
	}

	.tooltip-category {
		font-size: 0.65rem;
		font-weight: 600;
		text-transform: uppercase;
		letter-spacing: 0.08em;
		margin-bottom: 2px;
	}

	.tooltip-title {
		font-size: 0.85rem;
		font-weight: 600;
		color: #e5e5e5;
		line-height: 1.3;
	}

	.tooltip-hint {
		font-size: 0.6rem;
		color: rgba(255, 255, 255, 0.25);
		margin-top: 4px;
	}

	.detail-panel {
		position: fixed;
		right: 0;
		top: 0;
		bottom: 0;
		width: 360px;
		max-width: 90vw;
		background: rgba(10, 10, 15, 0.95);
		border-left: 1px solid rgba(255, 255, 255, 0.08);
		padding: 32px 24px;
		z-index: 100;
		overflow-y: auto;
		backdrop-filter: blur(20px);
		animation: slideIn 0.2s ease-out;
	}

	@keyframes slideIn {
		from {
			transform: translateX(100%);
			opacity: 0;
		}
		to {
			transform: translateX(0);
			opacity: 1;
		}
	}

	.close-btn {
		position: absolute;
		top: 16px;
		right: 16px;
		background: none;
		border: none;
		color: rgba(255, 255, 255, 0.4);
		font-size: 1.2rem;
		cursor: pointer;
		padding: 4px 8px;
	}

	.close-btn:hover {
		color: white;
	}

	.detail-category {
		font-size: 0.7rem;
		font-weight: 600;
		text-transform: uppercase;
		letter-spacing: 0.1em;
		margin-bottom: 4px;
	}

	.detail-year {
		font-size: 0.85rem;
		color: rgba(255, 255, 255, 0.4);
		margin-bottom: 12px;
	}

	.detail-title {
		font-size: 1.3rem;
		font-weight: 600;
		color: #e5e5e5;
		margin: 0 0 16px;
		line-height: 1.3;
	}

	.detail-desc {
		font-size: 0.9rem;
		color: rgba(255, 255, 255, 0.6);
		line-height: 1.6;
		margin: 0 0 24px;
	}

	.detail-significance {
		display: flex;
		align-items: center;
		gap: 4px;
	}

	.sig-dot {
		width: 8px;
		height: 8px;
		border-radius: 50%;
		background: rgba(255, 255, 255, 0.1);
	}

	.sig-dot.filled {
		background: var(--color-gold-400, #d4a843);
	}

	.sig-label {
		font-size: 0.7rem;
		color: rgba(255, 255, 255, 0.3);
		margin-left: 6px;
	}
</style>
