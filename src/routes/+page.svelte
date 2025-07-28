<script lang="ts">
	let multiplier = 1.0;
	let amount = 1.0;
	let history = [];

	let interval;
	let aviatorEl: HTMLImageElement;
	let graphEl: HTMLDivElement;

	let points: { x: number; y: number }[] = [];

	let x = 0;
	let y = 0;
	let angle = 0;
	let step = 0;
	let offsetY = 0;
	let offsetX = 0;
	let isFloating = false;
	let flyStart = true;

	let endMultiplier: number;

	function startGame() {
		multiplier = 1.0;
		history = [multiplier, ...history.slice(0, 14)];
		clearInterval(interval);
		step = 0;
		x = aviatorEl.x;
		y = aviatorEl.y;
		isFloating = false;
		flyStart = true;
		flyIn();

		endMultiplier = Math.random() * 5;

		interval = setInterval(() => {
			multiplier += Math.random() * 0.05;

			if (multiplier > endMultiplier) {
				endGame();
			}
		}, 100);
	}

	let isEnding = false;

	function endGame() {
		clearInterval(interval);
		isEnding = true;
		endFlyAway();
	}

	let trailFillPath;
	let trailStrokePath;
	let trailGroup;
	let scaleFactor = 1;

	let graphWrapper;

	function catmullRom2bezier(points) {
		if (points.length < 2) return '';

		let d = `M ${points[0].x},${points[0].y}`;

		for (let i = 0; i < points.length - 1; i++) {
			const p0 = points[i - 1] || points[i];
			const p1 = points[i];
			const p2 = points[i + 1];
			const p3 = points[i + 2] || p2;

			const cp1x = p1.x + (p2.x - p0.x) / 6;
			const cp1y = p1.y + (p2.y - p0.y) / 6;

			const cp2x = p2.x - (p3.x - p1.x) / 6;
			const cp2y = p2.y - (p3.y - p1.y) / 6;

			d += ` C ${cp1x},${cp1y} ${cp2x},${cp2y} ${p2.x},${p2.y}`;
		}

		return d;
	}

	function updatePolyline() {
		if (points.length < 2) return;

		const strokeD = catmullRom2bezier(points);

		// Заливка под линией
		const last = points[points.length - 1];
		const fillD = `${strokeD} L ${last.x},223 L 0,223 Z`;

		trailFillPath.setAttribute('d', fillD);
		trailStrokePath.setAttribute('d', strokeD);
	}

	function flyIn() {
		const startTime = performance.now();
		step = 0;
		const speed = 0.0009;
		const maxY =
			graphEl.getBoundingClientRect().height - aviatorEl.getBoundingClientRect().height - 20;

		function animateFlyIn(now) {
			if (isEnding) return;
			step += speed;

			// if (now - startTime > 1000) {
			// 	// Плавная параболическая траектория
			// 	x = step * 900;
			// 	y = -Math.pow(step * 300, 0.4); // Плавный взлёт, можешь поэкспериментировать с показателем
			// } else {
			// Плавная параболическая траектория
			x = step * 250;
			y = -Math.pow(x / 600, 2.1) * 1600; // Парабола: y = -k*x^2

			// }

			// Угол колебания
			angle = Math.sin(now / 1000) * 5;

			// Двигаем самолёт
			aviatorEl.style.transform = `translate(${x}px, ${y}px) rotate(${angle}deg)`;

			// Переводим координаты самолёта в SVG-систему
			const svgRect = trailFillPath.closest('svg').getBoundingClientRect();
			const planeRect = aviatorEl.getBoundingClientRect();

			const svgX =
				((planeRect.left + planeRect.width / 2 - svgRect.left) / svgRect.width) * 275 - 10;
			const svgY =
				((planeRect.top + planeRect.height / 2 - svgRect.top) / svgRect.height) * 223 + 10;

			points.push({ x: svgX, y: svgY });
			updatePolyline();

			// Продолжаем анимацию до достижения максимальной высоты
			if (-y < maxY) {
				requestAnimationFrame(animateFlyIn);
			} else {
				isFloating = true;
				floatInPlace();
			}
		}

		requestAnimationFrame(animateFlyIn);
	}

	function floatInPlace() {
		function animateFloat() {
			if (!isFloating) return;
			if (isEnding) return;

			offsetY = Math.sin(Date.now() / 500) * 5;
			offsetX = Math.sin(Date.now() / 700) * 3;
			angle = Math.sin(Date.now() / 300) * 3;

			aviatorEl.style.transform = `translate(${x + offsetX}px, ${y + offsetY}px) rotate(${angle}deg)`;

			// scaleFactor = Math.min(3, scaleFactor + 0.01);
			// trailGroup.setAttribute('transform', `scale(${scaleFactor})`);

			requestAnimationFrame(animateFloat);
		}

		animateFloat();
	}

	function endFlyAway() {
		const speedY = -3;
		const speedX = 2;

		function animateFlyAway() {
			if (!isEnding) return;

			y += speedY;
			x += speedX;

			angle = Math.sin(performance.now() / 300) * 5;

			aviatorEl.style.transform = `translate(${x}px, ${y}px) rotate(${angle}deg)`;

			// Обновляем линию
			const svgRect = trailFillPath.closest('svg').getBoundingClientRect();
			const planeRect = aviatorEl.getBoundingClientRect();

			const svgX = ((planeRect.left + planeRect.width / 2 - svgRect.left) / svgRect.width) * 275;
			const svgY = ((planeRect.top + planeRect.height / 2 - svgRect.top) / svgRect.height) * 223;

			requestAnimationFrame(animateFlyAway);
		}
		requestAnimationFrame(animateFlyAway);
	}
</script>

<div class="game-container">
	<!-- История -->
	<div class="history">
		{#each history as h}
			<span class:highlight={h > 5}>{h.toFixed(2)}x</span>
		{/each}
	</div>

	<!-- График -->

	<div class="graph" bind:this={graphEl}>
		<div class="graph-wrapper rotate" bind:this={graphWrapper}></div>
		<!-- <svg viewBox="0 0 200 100" preserveAspectRatio="none">
			<path d="M0,100 Q50,50 100,30 T200,0" fill="none" stroke="#0ff" stroke-width="2" />
		</svg> -->
		{#if multiplier > endMultiplier}
			<span class="end-text">Полетів</span>
		{/if}
		<span class={multiplier > endMultiplier ? 'multiplier end-multiplier' : 'multiplier'}
			>{multiplier.toFixed(2)}x</span
		>
		<svg class="trail" viewBox="0 0 275 220" preserveAspectRatio="none">
			<g bind:this={trailGroup}>
				<!-- Заливка -->
				<path bind:this={trailFillPath} fill="#008BB6" opacity="0.3" />
				<!-- Линия -->
				<path
					bind:this={trailStrokePath}
					stroke="#0ABAF0"
					stroke-width="3"
					stroke-linecap="round"
					fill="none"
				/>
			</g>
		</svg>

		<div class="plane">
			<img src="/images/aviator.png" alt="plane" bind:this={aviatorEl} />
		</div>
	</div>

	<div class="bets">
		<div class="sum">
			<div class="amount">
				<button on:click={() => (amount = Math.max(0.1, amount - 0.1))}>-</button>
				<span>{amount.toFixed(2)}</span>
				<button on:click={() => (amount += 0.1)}>+</button>
			</div>
			<div class="quick">
				{#each [1, 2, 5, 10] as val}
					<button on:click={() => (amount = val)}>{val}</button>
				{/each}
			</div>
		</div>
		<button class="bet" on:click={startGame}>Ставка {amount.toFixed(2)} USDT</button>
	</div>

	<button class="back">
		<img src="/images/arrow.png" alt="arrow img" />
		<span>Назад</span>
	</button>
</div>

<style>
	.game-container {
		background: #131313;
		color: white;
		font-family: sans-serif;
		padding: 20px;
		/* max-width: 400px; */
		margin: auto;
		min-height: 100vh;
		display: flex;
		flex-direction: column;
	}

	.history {
		display: flex;
		gap: 0.5rem;
		overflow-x: auto;
		margin-bottom: 1rem;
		padding-bottom: 0.5rem;
	}

	.history span {
		background: #222;
		padding: 0.3rem 0.6rem;
		border-radius: 12px;
		color: #00bfff;
		font-weight: bold;
		white-space: nowrap;
	}

	.history .highlight {
		color: #f0f;
	}

	.graph-wrapper {
		position: absolute;
		top: -40%;
		left: -150%;
		width: 300%;
		height: 300%;
		background: url('/images/bg.png') repeat;
		background-size: cover;
		transform-origin: center center;
		transform: rotate(0deg);
		transition: transform 0.2s ease-out;
		z-index: 0;
		pointer-events: none;
	}

	.graph {
		/* background: radial-gradient(#111 0%, #000 100%); */
		border-radius: 28px;
		padding: 4px;
		position: relative;
		overflow: hidden;
		min-height: 300px;
	}

	.trail {
		width: 100%;
		height: 300px;
		position: absolute;
		top: 0;
		left: 0;
	}

	.multiplier,
	.end-text {
		position: absolute;
		top: 50%;
		left: 50%;
		transform: translate(-50%, -50%);
		font-size: 82px;
		font-weight: bold;
		text-shadow: 0px 4px 72.7px #1276da;
		z-index: 2;
	}

	.end-text {
		top: 30%;
		font-size: 26px;
		font-weight: semibold;
	}

	.end-multiplier {
		color: #ff0000;
	}

	.plane {
		position: absolute;
		/* bottom: 5%; */
		bottom: 0;
		left: -3%;
		width: 70px;
		height: 60px;
	}

	.bets {
		display: flex;
		background-color: #212224;
		border-radius: 28px;
		gap: 12px;
		padding: 18px;
	}

	.sum {
		flex: 1 1 40%;
	}

	.amount {
		display: flex;
		justify-content: space-between;
		align-items: center;
		background-color: #191919;
		border-radius: 40px;
		gap: 20px;
	}

	.amount button {
		font-size: 20px;
		background-color: #3c3c3c;
		padding: 7px 18px;
		border-radius: 50%;
	}

	.quick {
		display: flex;
		flex-wrap: wrap;
		gap: 4px;
		margin-top: 8px;
	}

	.back {
		background: #009fdd;
		padding: 20px 50px;
		border-radius: 12px;
		font-size: 18px;
		font-weight: bold;
		border: none;
		display: flex;
		justify-content: center;
		align-items: center;
		margin-top: 80px;
	}

	.quick button {
		padding: 8px 0;
		font-size: 18px;
		color: #b3b3b3;
		background: #191919;
		border: none;
		border-radius: 40px;
		flex: 1 1 45%;
	}

	.bet {
		background: limegreen;
		color: white;
		padding: 1rem;
		border-radius: 10px;
		font-size: 1.2rem;
		font-weight: bold;
		border: none;
		flex: 1 1 60%;
	}

	@keyframes rotate-circle {
		0% {
			transform: rotate(0deg);
		}
		100% {
			transform: rotate(360deg);
		}
	}

	.graph-wrapper.rotate {
		animation: rotate-circle 30s linear infinite;
	}
</style>
