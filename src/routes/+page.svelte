<!-- <script>
	import { onMount } from 'svelte';

	let multiplier = 1.0;
	let points = [];

	let polylineEl;

	onMount(() => {
		const aviatorEl = document.querySelector('.aviator');
		const multiplierEl = document.querySelector('.multiplier');
		const backgroundEl = document.querySelector('.background');
		const sceneEl = document.querySelector('.scene');

		const maxY = window.innerHeight - parseFloat(aviatorEl.height) - window.innerHeight * 0.5;

		let x = 0;
		let y = 0;
		let angle = 0;
		let offsetY = 0;
		let offsetX = 0;
		let isFloating = false;
		let step = 1;

		let secondsPassed = 1;

		const aviatorElX = aviatorEl.x;
		const aviatorElY = aviatorEl.y;
		const aviatorElHeight = parseFloat(getComputedStyle(aviatorEl).height);

		setInterval(() => {
			secondsPassed++;
		}, 1000);

		setInterval(() => {
			if (multiplier < 10) multiplier *= 1 + Math.random() * 0.01 * secondsPassed;
			else multiplier *= 1 + Math.random() * 0.005 * secondsPassed;
		}, Math.random() * 75 + 25);

		function updatePolyline() {
			const path = points.map((p) => `${p.x},${p.y}`).join(' ');
			polylineEl.setAttribute('points', path);
		}

		function flyIn() {
			function animateFlyIn(now) {
				step += 0.002;
				x = step * 100 - 100;
				y = step * -200 + 200;
				angle = Math.sin(now / 1000) * 10;

				let start = true;

				setTimeout(() => {
					start = false;
				}, 1000);

				if (start) {
					x *= 2;
				}
				aviatorEl.style.transform = `translate(${x}px, ${y}px)`;
				points.push({ x: x + aviatorElX + 43, y: y + aviatorElY - aviatorElHeight * 2 });
				updatePolyline();

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
			let scale = 1;
			function animateFloat() {
				if (!isFloating) return;

				scale = Math.max(0.1, scale - 0.001);
				const time = Date.now();
				angle = Math.sin(time / 400) * 5;
				offsetY = Math.sin(time / 300) * 20;
				offsetX = Math.sin(time / 200) * 20;

				aviatorEl.style.transform = `translate(${x}px, ${y}px)`;
				sceneEl.style.transform = `scale(${scale})`;
				sceneEl.style.transformOrigin = 'center center';

				const aviatorBox = aviatorEl.getBoundingClientRect();

				// Компенсация scale
				const correctedX = (x + aviatorBox.width / 2) / scale;
				const correctedY = ((y + aviatorBox.height * 2) / scale) * -1;

				points.push({ x: correctedX, y: correctedY });

				updatePolyline();

				requestAnimationFrame(animateFloat);
			}

			animateFloat();
		}

		flyIn();
	});
</script>

<div class="world">
	<div class="scene">
		<div class="background"></div>
		<svg class="trail">
			<polyline bind:this={polylineEl} fill="none" stroke="lime" stroke-width="2" />
		</svg>
	</div>

	<img src="images/aviator.png" class="aviator" />
	<h1 class="multiplier">{multiplier.toFixed(2)}x</h1>
</div>

<style>
	body {
		margin: 0;
		overflow: hidden;
		background: #000;
	}

	.world {
		position: relative;
		width: 100vw;
		height: 100vh;
		overflow: hidden;
	}

	.scene {
		position: absolute;
		width: 100%;
		height: 100%;
		top: 0;
		left: 0;
		transform-origin: center center;
		transition: transform 0.2s ease-out;
	}

	.background {
		position: absolute;
		width: 10000px;
		height: 10000px;
		top: -4000px;
		left: -4000px;
		background: url('https://gmimages.cdnppb.net/paddypower-com/adabb193-440b-4766-ba52-8b8504412b05_DESIGNS-100925_Aviator_bg.jpg')
			no-repeat center center;
		background-size: cover;
		z-index: 0;
		transition: transform 0.2s ease-out;
		transform-origin: center center;
	}

	.aviator {
		width: 100px;
		filter: drop-shadow(0 0 10px lime);
		position: absolute;
		bottom: 50%;
		left: 1%;
		transform: translate(-50%, -50%);
		z-index: 2;
	}

	svg.trail {
		position: absolute;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		pointer-events: none;
		z-index: 1;
		background: rgba(0, 0, 0, 0.3); /* ✅ Фон под SVG */
		border: 1px solid lime;        /* ✅ Видимая граница */
		border-radius: 8px;            /* (необязательно) закругление углов */
		box-shadow: 0 0 10px lime;     /* (необязательно) свечение */
	}

	.multiplier {
		position: absolute;
		top: 10%;
		left: 50%;
		transform: translateX(-50%);
		color: white;
		font-size: 32px;
		font-family: monospace;
		z-index: 3;
		text-shadow: 0 0 5px #0f0;
	}
</style> -->

<script>
	let multiplier = 1.0;
	let amount = 1.0;
	let history = [];

	let interval;
	let aviatorEl;
	let graphEl;

	let points = [];

	let x = 0;
	let y = 0;
	let angle = 0;
	let step = 0;
	let offsetY = 0;
	let offsetX = 0;
	let isFloating = false;
	let flyStart = true;

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

		interval = setInterval(() => {
			multiplier += Math.random() * 0.05;
		}, 100);
	}

	let trailFillPath;
	let trailStrokePath;
	let trailGroup;
	let scaleFactor = 1;

	let graphWrapper;

	function updatePolyline() {
		if (points.length < 2) return;

		let d = `M ${points[0].x},${points[0].y}`;
		for (let i = 1; i < points.length; i++) {
			d += ` L ${points[i].x},${points[i].y}`;
		}

		// Заливка: продолжаем вниз к низу SVG и закрываем
		const last = points[points.length - 1];
		const fillD = `${d} L ${last.x},223 L 0,223 Z`;

		trailFillPath.setAttribute('d', fillD);
		trailStrokePath.setAttribute('d', d); // Только линия
	}

	function flyIn() {
		const rect = aviatorEl.getBoundingClientRect();
		const startTime = performance.now();

		let lastX;
		let lastY;
		let flyStartEndTime = 0;

		const maxY =
			graphEl.getBoundingClientRect().height - aviatorEl.getBoundingClientRect().height - 20;

		function animateFlyIn(now) {
			const elapsed = now - startTime;
			step += 0.0008;

			const graphRect = graphEl.getBoundingClientRect();

			let svgX = 0;
			let svgY = graphRect.height;

			points.push({ x: svgX, y: svgY });
			updatePolyline();

			// if (flyStart && elapsed < 2500) {
			// 	x = step * 350;
			// 	y = step * -200;
			// 	lastX = x;
			// 	lastY = y;
			// 	flyStartEndTime = now;
			// } else {
			// 	flyStart = false;
			// 	const flightElapsed = now - flyStartEndTime;
			// 	const flightStep = flightElapsed * 0.00015;

			// 	x = lastX + flightStep * 200;
			// 	y = lastY + flightStep * -250;
			// }

			x = step * 300;
			y = step * -300;

			angle = Math.sin(now / 1000) * 10;

			aviatorEl.style.transform = `translate(${x}px, ${y}px) rotate(${angle}deg)`;

			const svgRect = trailFillPath.closest('svg').getBoundingClientRect();
			const planeRect = aviatorEl.getBoundingClientRect();

			console.log(x, y);

			svgX = ((planeRect.left + planeRect.width / 2 - svgRect.left) / svgRect.width) * 275;
			svgY = ((planeRect.top + planeRect.height / 2 - svgRect.top) / svgRect.height) * 223;

			points.push({ x: svgX, y: svgY });
			updatePolyline();

			if (-y < maxY) {
				requestAnimationFrame(animateFlyIn);
			} else {
				// graphWrapper.classList.add('rotate');
				isFloating = true;
				floatInPlace();
			}
		}

		requestAnimationFrame(animateFlyIn);
	}

	function floatInPlace() {
		function animateFloat() {
			if (!isFloating) return;

			offsetY = Math.sin(Date.now() / 500) * 5;
			offsetX = Math.sin(Date.now() / 700) * 3;
			angle = Math.sin(Date.now() / 300) * 5;

			aviatorEl.style.transform = `translate(${x + offsetX}px, ${y + offsetY}px) rotate(${angle}deg)`;

			// scaleFactor = Math.min(3, scaleFactor + 0.01);
			// trailGroup.setAttribute('transform', `scale(${scaleFactor})`);

			requestAnimationFrame(animateFloat);
		}

		animateFloat();
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
		<span class="multiplier">{multiplier.toFixed(2)}x</span>
		<svg class="trail" viewBox="0 0 275 223" preserveAspectRatio="none">
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
		top: -165%;
		left: -165%;
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

	.multiplier {
		position: absolute;
		top: 50%;
		left: 50%;
		transform: translate(-50%, -50%);
		font-size: 82px;
		font-weight: bold;
		text-shadow: 0px 4px 72.7px #1276da;
		z-index: 2;
	}

	.plane {
		position: absolute;
		/* bottom: 5%; */
		bottom: 0;
		left: 1%;
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
