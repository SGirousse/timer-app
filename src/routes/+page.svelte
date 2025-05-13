<script lang="ts">
	import { Badge } from 'flowbite-svelte';
	let phases = $state([
		{ id: 1, name: 'planning', subphases: ['refresh', 'draw', 'draft', 'resources', 'haste'] },
		{
			id: 2,
			name: 'battle',
			subphases: [
				'attack',
				'pw1',
				'block/counter',
				'pw2',
				'combat',
				'pw3',
				'counterattack',
				'pw1',
				'block',
				'pw2',
				'combat',
				'pw3'
			]
		},
		{ id: 3, name: 'regroup', subphases: ['regroup'] },
		{ id: 4, name: 'deployment', subphases: ['deployment'] }
	]);
	let isGameActive = $state(false);
	let currentPhase = $state(phases[0]);
	let isAttackSkipped = $state(false);
	let isCounterAttackSkipped = $state(false);
	let TIME_INCREMENT = 30;
	let TIME_BASE = 180;

	let players = $state([
		{ id: 1, time: TIME_BASE, timer: null, subphase: 0 },
		{ id: 2, time: TIME_BASE, timer: null, subphase: 0 }
	]);
	let ITPlayer = $state(players[0]);
	let activePlayer = $state(players[0]);

	function resumeGame() {
		isGameActive = !isGameActive;
		resume(activePlayer);
	}

	function resume(player: any) {
		if (player.timer) {
			clearInterval(player.timer);
			player.timer = null;
		} else {
			player.timer = setInterval(() => {
				player.time--;
			}, 1000);
		}
	}

	function play(player: any) {
		resume(player);
		player.time += TIME_INCREMENT;

		// end of phase (everyone played)
		//FIXME wont work with more than 2 players
		if (ITPlayer !== player) {
			let isPhaseDone = false;
			let isTurnDone = false;

			if (currentPhase.name === 'planning') {
				if (player.subphase === phases.length) {
					isPhaseDone = true;
				}
			} else if (currentPhase.name === 'battle') {
				if (player.subphase === phases.length) {
					isPhaseDone = true;
				}
			} else if (currentPhase.name === 'regroup') {
				isPhaseDone = true;
			} else if (currentPhase.name === 'deployment') {
				isPhaseDone = true;
				isTurnDone = true;
			}
			console.log(isPhaseDone);
			console.log(currentPhase.name);

			if (isPhaseDone) {
				currentPhase = phases[currentPhase.id % phases.length];
				players.forEach((p) => {
					p.subphase = 0;
				});
			}

			if (isTurnDone) {
				ITPlayer = players[ITPlayer.id % players.length];
			}
		}

		player.subphase += 1;
		activePlayer = players[player.id % players.length];

		resume(activePlayer);
	}

	function skip(player: any) {
		resume(player);

		console.log(player.subphase);
		if (player.subphase === 0) {
			isAttackSkipped = true;
			players.forEach((p) => {
				p.subphase = 6;
			});
		} else if (player.subphase === 6) {
			players.forEach((p) => {
				p.subphase = 0;
			});

			if (isAttackSkipped) {
				currentPhase = phases[3]; // no battle : skip regroup and go to deployment
			} else {
				currentPhase = phases[2]; // battle occured : go to regroup
			}
		} else if (player.subphase in [1, 3, 6, 7, 9, 11]) {
		}
		activePlayer = players[player.id % players.length];

		resume(activePlayer);
	}

	function getCurrentSubphase() {
		return currentPhase.subphases[activePlayer.subphase];
	}
</script>

<main class="flex min-h-screen flex-col items-center justify-center bg-gray-100">
	<button
		onclick={resumeGame}
		class="mb-4 rounded bg-blue-500 px-4 py-2 text-white transition hover:bg-blue-600"
	>
		{isGameActive ? 'Pause Game' : 'Resume Game'}
	</button>

	<div class="flex space-x-2">
		<div class="flex flex-col items-center space-y-6">
			{#each players as player}
				<div
					class="w-80 rounded-lg p-6 text-center shadow-md {activePlayer === player
						? 'bg-green-100 text-green-800 dark:bg-green-900 dark:text-green-300'
						: 'bg-white'}"
				>
					<h2 class="mb-4 text-xl font-semibold">
						Player {player.id}
						{#if ITPlayer === player}(IT Player){/if}
					</h2>
					<p class="mb-4 text-lg">Time: <span class="font-mono">{player.time}</span> seconds</p>
					<button
						disabled={activePlayer !== player || !isGameActive}
						onclick={() => {
							play(player);
						}}
						class="rounded bg-blue-500 px-4 py-2 text-white transition hover:bg-blue-600 disabled:bg-gray-50 disabled:text-gray-500"
					>
						{#if !isGameActive}
							Game paused
						{:else if activePlayer === player}
							Action "{getCurrentSubphase()}" finished
						{:else}
							Waiting for opponent
						{/if}
					</button>
					<button
						disabled={activePlayer !== player || !isGameActive || currentPhase.name !== 'battle'}
						onclick={() => {
							skip(player);
						}}
						class="rounded bg-blue-500 px-4 py-2 text-white transition hover:bg-blue-600 disabled:bg-gray-50 disabled:text-gray-500"
					>
						Skip
					</button>
				</div>
			{/each}
		</div>
		<ul>
			{#each phases as phase}
				<li>
					<Badge rounded color={currentPhase === phase ? 'green' : 'gray'} class="font-bold">
						{phase.name.toUpperCase()}
					</Badge>

					<ul>
						{#each phase.subphases as subphase}
							<li>
								<Badge rounded color={getCurrentSubphase() === subphase ? 'green' : 'gray'}>
									{subphase}
								</Badge>
							</li>
						{/each}
					</ul>
				</li>
			{/each}
		</ul>
	</div>
</main>
