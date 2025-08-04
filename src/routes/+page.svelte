<script lang="ts">
	import { onMount } from 'svelte';
	import Matter from 'matter-js';

	let engine: Matter.Engine;
	let renderFrame: number;

	const multipliersEasy = [2, 1.5, 1.2, 1, 0.8, 1, 1.2, 1.5, 2];
	const probabilitiesEasy = [
		0.03, // 2x
		0.06, // 1.5x
		0.09, // 1.2x
		0.12, // 1x
		0.4, // 0.8x
		0.12, // 1x
		0.09, // 1.2x
		0.06, // 1.5x
		0.03 // 2x
	];

	const multipliersMedium = [75.5, 10, 3, 0.8, 0.3, 0.2, 0.3, 0.8, 3, 10, 75.5];
	const multipliersColors = [
		'linear-gradient(180deg, #FF7587 0%, #FF052B 100%)',
		'linear-gradient(180deg, #FF8175 0%, #FF1205 100%)',
		'linear-gradient(180deg, #FF8C75 0%, #FF3305 100%)',
		'linear-gradient(180deg, #FFA875 0%, #FF3F05 100%)',
		'linear-gradient(180deg, #FFB575 0%, #FF6905 100%)',
		'linear-gradient(180deg, #FFD075 0%, #FFAB05 100%)',
		'linear-gradient(180deg, #FFB575 0%, #FF6905 100%)',
		'linear-gradient(180deg, #FFA875 0%, #FF3F05 100%)',
		'linear-gradient(180deg, #FF8C75 0%, #FF3305 100%)',
		'linear-gradient(180deg, #FF8175 0%, #FF1205 100%)',
		'linear-gradient(180deg, #FF7587 0%, #FF052B 100%)'
	];
	const multipliersShadows = [
		'0px 4px 0px 0px #B80721',
		'0px 4px 0px 0px #B41006',
		'0px 4px 0px 0px #BF2A08',
		'0px 4px 0px 0px #BA360D',
		'0px 4px 0px 0px #BA530F',
		'0px 4px 0px 0px #A97713',
		'0px 4px 0px 0px #BA530F',
		'0px 4px 0px 0px #BA360D',
		'0px 4px 0px 0px #BF2A08',
		'0px 4px 0px 0px #B41006',
		'0px 4px 0px 0px #B80721'
	];
	const probabilitiesMedium = [
		0.001, // 75.5
		0.005, // 10
		0.03, // 3
		0.114, // 0.8
		0.16, // 0.3
		0.38, // 0.2
		0.16, // 0.3
		0.114, // 0.8
		0.03, // 3
		0.005, // 10
		0.001 // 75.5
	];

	let historyColors: string[] = [];

	$: historyColors = lastFiveWinCoeffs.map((historyItem) => {
		if (historyItem === 75.5) return 'linear-gradient(180deg, #FF7587 0%, #FF052B 100%)';
		if (historyItem === 10) return 'linear-gradient(180deg, #FF8175 0%, #FF1205 100%)';
		if (historyItem === 3) return 'linear-gradient(180deg, #FF8C75 0%, #FF3305 100%)';
		if (historyItem === 0.8) return 'linear-gradient(180deg, #FFA875 0%, #FF3F05 100%)';
		if (historyItem === 0.3) return 'linear-gradient(180deg, #FFB575 0%, #FF6905 100%)';
		return 'linear-gradient(180deg, #FFD075 0%, #FFAB05 100%)';
	});

	let betSum = 1;
	let winSum = 0;

	console.log(probabilitiesMedium.reduce((acc, prob) => acc + prob, 0));

	const rtp = multipliersMedium.reduce((acc, mult, i) => acc + mult * probabilitiesMedium[i], 0);
	console.log('RTP:', rtp); // ~0.98

	const multipliersHard = [100, 10, 5, 2, 0, 2, 5, 10, 100];
	const probabilitiesHard = [
		0.005, // 100x
		0.01, // 10x
		0.025, // 5x
		0.05, // 2x
		0.82, // 0x
		0.05, // 2x
		0.025, // 5x
		0.01, // 10x
		0.005 // 100x
	]; // сумма = 1.0

	let multipliers = multipliersMedium;
	let probabilities = probabilitiesMedium;

	const minCols = 3;
	const maxCols = 12;
	const spacing = 32; // important
	const rows = maxCols - minCols + 1;

	// for counting rtp
	let totalWins = 0;
	let totalPlays = 0;
	let averageMultiplier = 0;
	const slotCount = multipliers.length;
	const slotWidth = spacing;

	let lastFiveWinCoeffs = [];

	let jumpingMultipliers = Array(multipliers.length).fill(false);

	let forceMultiplier = 0.00001;
	let offsetX = 0;

	let allBalls: { id: number; body: Matter.Body; targetX: number }[] = [];
	let nextBallId = 0;

	function getColsForRow(row: number) {
		return Math.min(minCols + row, maxCols);
	}

	function chooseTargetIndex(): number {
		const r = Math.random();
		let acc = 0;

		let candidates: number[] = [];
		let candidateProb = 0;

		for (let i = 0; i < probabilities.length; i++) {
			acc += probabilities[i];
			if (r <= acc) {
				// Найдём все индексы с такой же вероятностью
				candidateProb = probabilities[i];
				candidates = probabilities
					.map((p, idx) => (p === candidateProb ? idx : -1))
					.filter((idx) => idx !== -1);
				break;
			}
		}

		if (candidates.length > 0) {
			const randomIdx = Math.floor(Math.random() * candidates.length);
			console.log(candidates, randomIdx);
			if (randomIdx === 1) {
				forceMultiplier = 0.00002;
				offsetX = Math.random() * spacing * 0.1;
			} else {
				offsetX = -1 * Math.random() * spacing * 0.1;
			}
			if (candidates.length === 1) {
				offsetX = Math.random() * spacing * 0.1 - spacing * 0.05;
			}
			return candidates[randomIdx];
		}

		// fallback — если вдруг не сработало
		return probabilities.length - 1;
	}

	function dropBall() {
		const targetIndex = chooseTargetIndex();
		const startY = -20;
		const ballRadius = spacing / 2 / 1.6;

		const center = Math.floor(maxCols / 2);
		const startX = center * spacing + offsetX;
		// const startX = center * spacing;

		const newBall = Matter.Bodies.circle(startX, startY, ballRadius, {
			// label: 'ball',
			// restitution: 0.9,
			// density: 0.06,
			// friction: 0.2,
			// frictionAir: 0.25,
			// collisionFilter: {
			// 	group: -1
			// }
			restitution: 0.1, // тоже упругий
			density: 0.06,
			friction: 0.2,
			frictionAir: 0.1,
			collisionFilter: {
				group: -1
			},
			label: 'ball'
		});

		const targetX =
			targetIndex < center
				? targetIndex * spacing + spacing / 2
				: (targetIndex + 1) * spacing + spacing / 2;

		Matter.World.add(engine.world, newBall);
		allBalls = [...allBalls, { id: nextBallId++, body: newBall, targetX }];

		setTimeout(() => {
			let localForce = forceMultiplier;
			const forceInterval = setInterval(() => {
				localForce = Math.min(forceMultiplier, localForce + 0.000001);
			}, 30);

			const attractor = () => {
				if (newBall.position.y < (rows - 1) * spacing) {
					const dy = newBall.position.y;
					const time = performance.now() / 1000;

					if (!newBall['intermediateTargetX']) {
						const range = spacing * 6;
						newBall['intermediateTargetX'] =
							newBall.position.x + (Math.random() * range - range / 2);
					}
					const intermediateX = newBall['intermediateTargetX'];

					// Прогресс — на сколько глубоко шарик прошёл
					const progress = Math.min(dy / ((rows - 1) * spacing), 1);

					// Более "плавное" приближение: easeInOut
					const smoothProgress = 0.5 * (1 - Math.cos(Math.PI * progress));

					const dynamicTargetX = intermediateX + (targetX - intermediateX) * smoothProgress;

					const dx = dynamicTargetX - newBall.position.x;
					const dxNorm = Math.min(Math.abs(dx) / spacing, 1.5);

					// Уменьшаем дрожание
					const oscillation = Math.sin(time * 2.5 + newBall.id) * 0.08;

					// Новый коэффициент притяжения
					const magnetBoost = 1 + 2.2 * smoothProgress + 1.2 * dxNorm;

					// Более мягкое усиление на краях
					const centerIndex = (slotCount - 1) / 2;
					const edgeBias = 1 + Math.pow(Math.abs(targetIndex - centerIndex), 1.1) * 0.08;

					const forceX = (dx + oscillation * spacing) * localForce * magnetBoost * edgeBias;

					Matter.Body.applyForce(newBall, newBall.position, { x: forceX, y: 0 });
				} else {
					Matter.Events.off(engine, 'beforeUpdate', attractor);
					clearInterval(forceInterval);
				}
			};

			setTimeout(() => {
				Matter.Events.on(engine, 'beforeUpdate', attractor);
			}, 70);

			if (!renderFrame) {
				update();
			}
		}, 700);
	}

	function update() {
		Matter.Engine.update(engine);

		for (let i = allBalls.length - 1; i >= 0; i--) {
			const ball = allBalls[i];
			const { x, y } = ball.body.position;

			if (y > (rows + 0.65) * spacing) {
				// console.log(x, slotWidth);
				const finalIndex = Math.min(slotCount - 1, Math.max(0, Math.floor((x - 3) / slotWidth)));
				// console.log('finalIndex', finalIndex);
				totalWins += multipliers[finalIndex];
				winSum = betSum * multipliers[finalIndex];
				totalPlays += 1;
				averageMultiplier = +(totalWins / totalPlays).toFixed(3);
				pegs = pegs.map((p) => ({ ...p, highlighted: false }));
				Matter.World.remove(engine.world, ball.body);
				allBalls.splice(i, 1);

				jumpingMultipliers[finalIndex] = true;
				setTimeout(() => {
					jumpingMultipliers[finalIndex] = false;
				}, 300);

				lastFiveWinCoeffs.push(multipliers[finalIndex]);
				if (lastFiveWinCoeffs.length > 5) {
					lastFiveWinCoeffs.shift();
				}
				lastFiveWinCoeffs = [...lastFiveWinCoeffs];

				console.log(totalWins, totalPlays, averageMultiplier);
			}
		}

		allBalls = [...allBalls];

		if (allBalls.length > 0) {
			renderFrame = requestAnimationFrame(update);
		} else {
			cancelAnimationFrame(renderFrame);
			renderFrame = null;
		}
	}

	function highlightPeg(label: string) {
		const peg = pegs.find((p) => p.label === label);
		if (peg) {
			peg.highlighted = true;
			setTimeout(() => {
				peg.highlighted = false;
			}, 50);
		}
	}

	function changeDifficulty(difficulty: 'easy' | 'medium' | 'hard') {
		switch (difficulty) {
			case 'easy':
				multipliers = multipliersEasy;
				probabilities = probabilitiesEasy;
				break;
			case 'medium':
				multipliers = multipliersMedium;
				probabilities = probabilitiesMedium;
				break;
			case 'hard':
				multipliers = multipliersHard;
				probabilities = probabilitiesHard;
				break;
			default:
				multipliers = multipliersMedium;
		}
	}

	let pegs: { label: string; x: number; y: number; highlighted: boolean }[] = [];

	for (let rowIdx = 0; rowIdx < rows; rowIdx++) {
		const cols = getColsForRow(rowIdx);
		for (let colIdx = 0; colIdx < cols; colIdx++) {
			const x = colIdx * spacing + spacing / 2.2 + ((maxCols - cols) * spacing) / 2;
			const y = rowIdx * spacing + spacing;
			const label = `peg-${rowIdx}-${colIdx}`;
			pegs.push({ label, x, y, highlighted: false });
		}
	}

	onMount(() => {
		engine = Matter.Engine.create();

		const pegRadius = spacing / 5;

		pegs.forEach(({ x, y, label }) => {
			const pegBody = Matter.Bodies.circle(x, y, pegRadius, {
				isStatic: true,
				restitution: 1, // Упругость — 1.0 = идеально упругое тело
				friction: 0.1,
				frictionAir: 0.001,
				label: 'peg'
			});
			Matter.World.add(engine.world, pegBody);
		});

		Matter.Events.on(engine, 'collisionStart', (event) => {
			for (const pair of event.pairs) {
				const { bodyA, bodyB } = pair;

				let pegBody = null;
				if (bodyA.label === 'ball' && bodyB.label.startsWith('peg')) pegBody = bodyB;
				if (bodyB.label === 'ball' && bodyA.label.startsWith('peg')) pegBody = bodyA;

				if (pegBody) {
					pegs = [...pegs];
					highlightPeg(pegBody.label);
				}
			}
		});
	});

	function changeBet(operation: string) {
		if (operation === 'minus') {
			if (betSum >= 0.2) {
				betSum -= 0.1;
			}
		} else {
			betSum += 0.1;
		}
	}
</script>

<div class="container">
	<div class="board" style={`--rows: ${rows}; --spacing: ${spacing}px; --maxCols: ${maxCols};`}>
		<aside class="board__aside">
			{#each lastFiveWinCoeffs.slice().reverse() as coeff}
				<div
					class="board__aside-coeff"
					style="background: {historyColors[lastFiveWinCoeffs.indexOf(coeff)]};"
				>
					{coeff.toFixed(2)}x
				</div>
			{/each}
		</aside>

		<!-- {#each Array(rows) as _, rowIdx}
		{#each Array(getColsForRow(rowIdx)) as _, colIdx}
			<div
				data-id={`peg-${rowIdx}-${colIdx}`}
				class="board__peg"
				style={`top: ${rowIdx * spacing + spacing}px; left: ${colIdx * spacing + spacing / 2.2 + ((maxCols - getColsForRow(rowIdx)) * spacing) / 2}px`}
			></div>
		{/each}
	{/each} -->

		<div class="board__peg board__peg--start"></div>
		{#each pegs as peg}
			<div
				data-id={peg.label}
				class="board__peg"
				class:highlight={peg.highlighted}
				style={`top: ${peg.y}px; left: ${peg.x}px;`}
			></div>
		{/each}

		{#each allBalls as ball (ball.id)}
			<div
				class="board__ball transparent-animation"
				style={`transform: translate(${ball.body.position.x}px, ${ball.body.position.y}px) translate(-50%, -50%)`}
			></div>
		{/each}
		<div class="board__multipliers">
			{#each multipliers as multiplier, idx}
				<div
					class="board__multiplier"
					class:jump={jumpingMultipliers[idx]}
					style={`left: ${idx * spacing + spacing - 1}px; background: ${multipliersColors[idx]}; width: ${spacing * 0.97}px; box-shadow: ${multipliersShadows[idx]};`}
				>
					<span class="board__multiplier__text">{multiplier}</span>
					<span class="board__multiplier__text">x</span>
					<div class="bg-win" class:active={jumpingMultipliers[idx] && (idx < 3 || idx > 7)}></div>
				</div>
			{/each}
		</div>
	</div>

	<div class="score">
		Виграш {winSum.toFixed(2)} USDT
	</div>

	<div class="bets">
		<button class:disabled={betSum < 0.2} on:click={() => changeBet('minus')}
			><img src="./images/minus-button.png" alt="minus button image" /></button
		>
		<button class="bets__button">Ставка: {betSum.toFixed(2)} USDT</button>
		<button on:click={() => changeBet('plus')}
			><img src="./images/plus-button.png" alt="plus button image" /></button
		>
	</div>

	<!-- <div class="board__buttons">
	<div class="board__difficulty">
		<button class="board__difficulty__button" on:click={() => changeDifficulty('easy')}>Easy</button
		>
		<button class="board__difficulty__button" on:click={() => changeDifficulty('medium')}
			>Medium</button
		>
		<button class="board__difficulty__button" on:click={() => changeDifficulty('hard')}>Hard</button
		>
	</div>
	<button class="play-button" on:click={dropBall}
		><img src="./images/button.png" alt="button icon" /></button
	>
</div> -->

	<button class="play-button" on:click={dropBall}
		><img src="./images/button.png" alt="button icon" /></button
	>
</div>

<style>
	.container {
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		padding: 0 10px;
	}

	.board {
		position: relative;
		width: calc(var(--maxCols) * var(--spacing));
		height: calc(var(--rows) * var(--spacing) + var(--spacing));
		margin: 0 auto;
		margin-top: 150px;
	}

	.board__aside {
		width: 80px;
		height: 200px;
		border: 1px solid #ffffff;
		background: #00193e;
		border-radius: 12px;
		position: absolute;
		z-index: 10;
		top: -15%;
		overflow: hidden;
	}

	.board__aside-coeff {
		font-weight: 800;
		font-size: 16px;
		letter-spacing: 2%;
		text-align: center;
		-webkit-text-stroke: 0.5px solid #00000080;
		/* background: linear-gradient(180deg, #ffd075 0%, #ffab05 100%); */
		color: #ffffff;
		height: 20%;
		display: flex;
		justify-content: center;
		align-items: center;
	}

	.board__peg {
		position: absolute;
		transform: translate(-50%, -50%);
		width: calc(var(--spacing) / 2.5);
		height: calc(var(--spacing) / 2.5);
		background: #fff;
		border-radius: 50%;
	}

	.board__peg--start {
		background-color: #000000;
		box-shadow: 0px 0px 14px 0px #ffffff40;
		left: 50%;
		top: -20px;
		transform: translate(-50%, -50%);
		width: calc(var(--spacing));
		height: calc(var(--spacing));
		border-radius: 50%;
	}

	.board__ball {
		position: absolute;
		width: calc(var(--spacing) / 1.7);
		height: calc(var(--spacing) / 1.7);
		background: url('./images/ball.png');
		border-radius: 50%;
		z-index: 10;
	}

	.board__buttons {
		display: flex;
		justify-content: center;
		flex-direction: column;
	}

	.board__difficulty {
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 10px;
	}

	.board__difficulty__button {
		padding: 10px 20px;
		font-size: 20px;
		background: #67b640;
		color: #fff;
		cursor: pointer;
		border-radius: 10px;
		width: 200px;
	}

	.score {
		margin: 65px auto 0;
		text-align: center;
		background: url('./images/winBg.png');
		height: 40px;
		display: flex;
		justify-content: center;
		align-items: center;
		color: #fff;
		font-size: 22px;
		font-weight: 700;
		border-radius: 8px;
		width: 100%;
		box-shadow: 0px 0px 10px 0px #fec801;
		color: #433600;
	}

	.play-button {
		display: block;
		margin: 20px auto;
		cursor: pointer;
	}

	.play-button:active {
		filter: brightness(0.9);
	}

	.board__multipliers {
		position: absolute;
		bottom: -10px;
		width: 100%;
		height: 20px;
		pointer-events: none;
	}

	.board__multiplier {
		position: absolute;
		transform: translateX(-50%);
		text-align: center;
		font-weight: bold;
		color: #fff;
		font-size: 13px;
		font-weight: 800;
		display: flex;
		flex-direction: column;
		align-items: center;
		border-radius: 5px;
	}

	.board__multiplier__text {
		-webkit-text-stroke: 0.5px #00000080;
	}

	.bg-win {
		position: absolute;
		left: 0%;
		top: -170%;
		transform: translateX(-50%);
		width: 100%;
		height: 100px;
		pointer-events: none;
		opacity: 0;
		transform: rotate(180deg);
		background: linear-gradient(180deg, rgba(255, 113, 82, 0.7) 0%, rgba(255, 113, 82, 0) 100%);
		transition: opacity 0.2s ease;
		z-index: -1;
	}

	.bg-win.active {
		opacity: 1;
		animation: bigWin 0.5s ease-out;
	}

	@keyframes bigWin {
		0% {
			transform: scaleY(0.7), rotate(180deg);
			opacity: 0;
		}
		50% {
			transform: scaleY(1.2), rotate(180deg);
			opacity: 1;
		}
		100% {
			transform: scaleY(1), rotate(180deg);
			opacity: 0;
		}
	}

	.board__peg.highlight {
		box-shadow: 0px 0px 4px 0px #ff0073;
		border: 2px solid #ff1b73b2;
	}

	@keyframes transparent {
		0% {
			opacity: 0;
			top: -10px;
		}
		100% {
			opacity: 1;
			top: 0;
		}
	}

	.transparent-animation {
		animation: 0.5s transparent ease-in-out;
	}

	.board__multiplier.jump {
		animation: bounce 0.3s ease;
	}

	@keyframes bounce {
		0% {
			transform: translate(-50%, 0);
		}
		50% {
			transform: translate(-50%, -10px);
		}
		100% {
			transform: translate(-50%, 0);
		}
	}

	.bets {
		display: flex;
		justify-content: center;
		gap: 6px;
		margin-top: 15px;
	}

	.bets button {
		cursor: pointer;
	}

	.bets__button {
		background: #55b64233;
		font-weight: 700;
		font-size: 18px;
		letter-spacing: 2%;
		text-align: center;
		padding: 12px 16px;
		color: #f0ffef;
	}

	.disabled {
		pointer-events: none;
		opacity: 0.5;
	}
</style>
