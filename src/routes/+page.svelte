<script lang="ts">
	import { onMount } from 'svelte';

	type Pipe = { x: number; gap: number; scored: boolean };
	type Status = 'ready' | 'playing' | 'over';

	const W = 400;
	const H = 600;
	const FLOOR = 550;
	const PIPE_W = 64;
	const GAP = 155;

	let canvas: HTMLCanvasElement;
	let status = $state<Status>('ready');
	let score = $state(0);
	let best = $state(0);
	let y = 280;
	let velocity = 0;
	let pipes: Pipe[] = [];
	let nextPipe = 0;
	let frame = 0;

	function start() {
		if (status !== 'playing') {
			status = 'playing';
			score = 0;
			y = 280;
			velocity = 0;
			pipes = [];
			nextPipe = 0.7;
		}
		velocity = -350;
	}

	function update(dt: number) {
		if (status !== 'playing') return;
		velocity += 980 * dt;
		y += velocity * dt;
		nextPipe -= dt;

		if (nextPipe <= 0) {
			pipes.push({ x: W, gap: 90 + Math.random() * 260, scored: false });
			nextPipe = 1.5;
		}

		for (const pipe of pipes) {
			pipe.x -= 135 * dt;
			if (!pipe.scored && pipe.x + PIPE_W < 100) {
				pipe.scored = true;
				score += 1;
			}
		}
		pipes = pipes.filter((pipe) => pipe.x > -PIPE_W);

		const hitPipe = pipes.some(
			(pipe) =>
				124 > pipe.x && 78 < pipe.x + PIPE_W && (y - 15 < pipe.gap || y + 15 > pipe.gap + GAP)
		);
		if (y < 15 || y > FLOOR - 15 || hitPipe) {
			status = 'over';
			best = Math.max(best, score);
			localStorage.setItem('flappy-fish-best', String(best));
		}
	}

	function draw(ctx: CanvasRenderingContext2D, time: number) {
		const ocean = ctx.createLinearGradient(0, 0, 0, H);
		ocean.addColorStop(0, '#087ca0');
		ocean.addColorStop(1, '#55cab2');
		ctx.fillStyle = ocean;
		ctx.fillRect(0, 0, W, H);

		ctx.strokeStyle = '#b8f4e766';
		ctx.lineWidth = 2;
		for (let i = 0; i < 10; i += 1) {
			ctx.beginPath();
			ctx.arc((i * 83) % W, (i * 91 - time / 35 + 700) % FLOOR, 3 + (i % 3) * 2, 0, Math.PI * 2);
			ctx.stroke();
		}

		for (const pipe of pipes) {
			ctx.fillStyle = '#08776c';
			ctx.fillRect(pipe.x, 0, PIPE_W, pipe.gap);
			ctx.fillRect(pipe.x, pipe.gap + GAP, PIPE_W, FLOOR);
			ctx.fillStyle = '#20aa91';
			ctx.fillRect(pipe.x - 5, pipe.gap - 20, PIPE_W + 10, 20);
			ctx.fillRect(pipe.x - 5, pipe.gap + GAP, PIPE_W + 10, 20);
		}

		ctx.save();
		ctx.translate(100, y + (status === 'ready' ? Math.sin(time / 250) * 6 : 0));
		ctx.rotate(status === 'playing' ? Math.max(-0.3, Math.min(0.6, velocity / 650)) : 0);
		ctx.fillStyle = '#e95f3a';
		ctx.beginPath();
		ctx.moveTo(-18, 0);
		ctx.lineTo(-36, -14);
		ctx.lineTo(-34, 14);
		ctx.fill();
		ctx.fillStyle = '#ffb532';
		ctx.beginPath();
		ctx.ellipse(0, 0, 25, 18, 0, 0, Math.PI * 2);
		ctx.fill();
		ctx.fillStyle = 'white';
		ctx.beginPath();
		ctx.arc(15, -6, 7, 0, Math.PI * 2);
		ctx.fill();
		ctx.fillStyle = '#123b50';
		ctx.beginPath();
		ctx.arc(17, -6, 3, 0, Math.PI * 2);
		ctx.fill();
		ctx.restore();

		ctx.fillStyle = '#f2ca6c';
		ctx.fillRect(0, FLOOR, W, H - FLOOR);
	}

	onMount(() => {
		best = Number(localStorage.getItem('flappy-fish-best') ?? 0);
		const ctx = canvas.getContext('2d');
		if (!ctx) return;
		let previous = 0;

		const loop = (time: number) => {
			update(Math.min((time - previous) / 1000 || 0, 0.035));
			previous = time;
			draw(ctx, time);
			frame = requestAnimationFrame(loop);
		};
		const keyboard = (event: KeyboardEvent) => {
			if (event.code === 'Space' || event.code === 'ArrowUp') {
				event.preventDefault();
				start();
			}
		};

		window.addEventListener('keydown', keyboard);
		frame = requestAnimationFrame(loop);
		return () => {
			window.removeEventListener('keydown', keyboard);
			cancelAnimationFrame(frame);
		};
	});
</script>

<svelte:head><title>Flappy Fish</title></svelte:head>

<main>
	<header>
		<h1>Flappy Fish</h1>
		<span>Best: {best}</span>
	</header>
	<div class="game">
		<canvas bind:this={canvas} width={W} height={H} onclick={start} aria-label="Flappy Fish game"
		></canvas>
		{#if status === 'playing'}
			<strong class="score">{score}</strong>
		{:else}
			<section class="card">
				<h2>{status === 'ready' ? 'Ready to dive?' : `Score: ${score}`}</h2>
				<p>{status === 'ready' ? 'Swim through the coral pipes.' : 'Give it another go!'}</p>
				<button onclick={start}>{status === 'ready' ? 'Play' : 'Play again'}</button>
			</section>
		{/if}
	</div>
	<p class="hint">Click, tap, or press Space to swim</p>
</main>

<style>
	:global(*) {
		box-sizing: border-box;
	}
	:global(body) {
		margin: 0;
		background: #07394d;
		color: white;
		font-family: system-ui, sans-serif;
	}
	main {
		width: min(440px, calc(100% - 24px));
		margin: auto;
		padding: 24px 0;
	}
	header {
		display: flex;
		align-items: center;
		justify-content: space-between;
	}
	h1 {
		margin: 0 0 14px;
		font-size: 1.5rem;
	}
	header span {
		color: #a9dcd9;
		font-weight: 700;
	}
	.game {
		position: relative;
		overflow: hidden;
		aspect-ratio: 2 / 3;
		border: 5px solid #062b3a;
		border-radius: 24px;
		box-shadow: 0 20px 50px #001d2d88;
	}
	canvas {
		display: block;
		width: 100%;
		height: 100%;
		cursor: pointer;
		touch-action: manipulation;
	}
	.score {
		position: absolute;
		top: 24px;
		left: 0;
		width: 100%;
		text-align: center;
		font-size: 3rem;
		text-shadow: 0 3px #075060;
		pointer-events: none;
	}
	.card {
		position: absolute;
		top: 50%;
		left: 50%;
		width: 75%;
		padding: 24px;
		border-radius: 18px;
		background: #07394ddd;
		text-align: center;
		transform: translate(-50%, -50%);
		backdrop-filter: blur(6px);
	}
	.card h2 {
		margin: 0 0 8px;
	}
	.card p {
		margin: 0 0 20px;
		color: #b8dfdc;
	}
	button {
		width: 100%;
		padding: 13px;
		border: 0;
		border-radius: 10px;
		background: #FF2800;
		color: #07394d;
		font: inherit;
		font-weight: 800;
		cursor: pointer;
	}
	button:focus-visible {
		outline: 3px solid white;
		outline-offset: 3px;
	}
	.hint {
		margin: 12px 0 0;
		color: #9bc9ca;
		text-align: center;
		font-size: 0.8rem;
	}
</style>
