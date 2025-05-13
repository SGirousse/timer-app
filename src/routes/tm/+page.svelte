<script lang="ts">
	import {
		PauseOutline,
		PlayOutline,
		CheckCircleOutline,
		RocketOutline,
		CloseCircleOutline,
		AwardOutline,
		ClockOutline,
		UserAddOutline,
		UserRemoveOutline,
		RocketSolid
	} from 'flowbite-svelte-icons';
	import { Toast } from 'flowbite-svelte';

	let DEFAULT_TOAST_TIMEOUT = 3;
	let toastStatus = $state(false);
	let toastMsg = $state('');
	let toastTimeoutInS = $state(DEFAULT_TOAST_TIMEOUT);

	let CLOCK_TYPES = [
		{ name: 'Fisher', baseTimeInS: 600, incrementTimeInS: 300 },
		{ name: 'Mort subite', baseTimeInS: 3000, incrementTimeInS: 0 }
	];

	let PHASES = ['Mise en place', 'Recherche', 'Actions'];

	let isGameStarted = $state(false);
	let turn = $state(1);
	let clocktype = $state(CLOCK_TYPES[0]);
	let currentPhase = $state(PHASES[0]);

	let players = $state([
		{
			name: 'Player 1',
			time: 0,
			timer: null as number | null,
			hasPassed: false,
			isActive: false
		},
		{
			name: 'Player 2',
			time: 0,
			timer: null as number | null,
			hasPassed: false,
			isActive: false
		}
	]);

	function addPlayer() {
		players = [
			...players,
			{
				name: 'Player',
				time: 0,
				timer: null as number | null,
				hasPassed: false,
				isActive: false
			}
		];
	}

	function removePlayer(player: any) {
		players = players.filter((p) => p !== player);
	}

	let firstPlayer = $state(null as any);

	let isGameActive = $state(false);

	function nextPhase() {
		if (isGameStarted) {
			if (currentPhase == PHASES[0]) {
				currentPhase = PHASES[1];
			} else {
				isGameActive = !isGameActive;
			}
			players.forEach((player) => resumePlayer(player));
		} else {
			isGameStarted = true;
			isGameActive = true;
			firstPlayer = players[Math.floor(Math.random() * players.length)];
			players.forEach((player) => (player.time = clocktype.baseTimeInS));
			postToast(`Début de la Terraformation !`);
		}
	}

	function resumeGame() {
		isGameActive = !isGameActive;
		players.forEach((player) => resumePlayer(player, false));
	}

	function resumePlayer(player: any, updateStatus: boolean = true) {
		if (updateStatus) {
			player.isActive = !player.isActive;
		}

		if (player.timer || !isGameActive) {
			clearInterval(player.timer);
			player.timer = null;
		} else if (player.isActive) {
			player.timer = setInterval(() => {
				player.time--;
			}, 1000);
		}
	}

	function play(player: any) {
		// Pause the active player
		resumePlayer(player);

		// Move to next player (could be themselves)
		let nextIndex = (players.indexOf(player) + 1) % players.length;

		while (players[nextIndex].hasPassed) {
			nextIndex = (nextIndex + 1) % players.length;
		}

		let nextPlayer = players[nextIndex];

		// Start the player timer
		resumePlayer(nextPlayer);
	}

	function pass(player: any) {
		let isTurnEnded = false;

		// Pause the active player
		player.hasPassed = true;
		resumePlayer(player);

		if (currentPhase === PHASES[1]) {
			// RESEARCH : everything is parallel
			// Check is the pass player was the last one, if so, go to the ACTION PHASE
			const anyActivePlayer = players.find((player) => player.hasPassed === false);
			if (!anyActivePlayer) {
				players.forEach((player) => (player.hasPassed = false));
				currentPhase = PHASES[2];
				resumePlayer(firstPlayer);
			}
		} else if (currentPhase === PHASES[2]) {
			player.time += clocktype.incrementTimeInS;

			// ACTION : each player is predefined order

			// Move to next player if available
			let nextIndex = (players.indexOf(player) + 1) % players.length;

			while (players[nextIndex].hasPassed && !isTurnEnded) {
				if (nextIndex === players.indexOf(player)) {
					isTurnEnded = true;
				} else {
					nextIndex = (nextIndex + 1) % players.length;
				}
			}

			if (isTurnEnded) {
				// If everyone has passed, then end the turn and go to the next
				nextTurn();
			} else {
				// Set the next player as active
				let nextPlayer = players[nextIndex];
				resumePlayer(nextPlayer);
			}
		}
	}

	function nextTurn() {
		// Reset players pass status
		players.forEach((player) => (player.hasPassed = false));

		// Reset to first phase
		currentPhase = PHASES[0];

		// Update player order
		firstPlayer = players[(players.indexOf(firstPlayer) + 1) % players.length];

		turn += 1;

		postToast(`Début du tour ${turn}`);
	}

	function postToast(message: string) {
		toastMsg = message;
		toastTimeoutInS = DEFAULT_TOAST_TIMEOUT;
		toastStatus = true;
		toastTimeout();
	}

	function toastTimeout() {
		if (--toastTimeoutInS > 0) return setTimeout(toastTimeout, 1000);
		toastStatus = false;
	}
</script>

<div class="mr-1 ml-1 flex h-screen flex-col items-center justify-center bg-gray-100">
	<div class="flex h-1/6 items-center justify-center">
		<Toast
			contentClass="flex items-center justify-center space-x-4 rtl:space-x-reverse divide-x rtl:divide-x-reverse divide-gray-200 dark:divide-gray-700"
			bind:toastStatus
		>
			{#snippet icon()}
				<RocketSolid class="text-primary-600 dark:text-primary-500 h-5 w-5 rotate-45" />
			{/snippet}
			<div class="text-sm font-normal">{toastMsg}</div>
		</Toast>
	</div>
	<div class="flex h-full w-full flex-col items-center space-y-6 md:w-1/2">
		<div class="w-full rounded-lg bg-white p-6 text-center shadow-md">
			<div class="flex items-center justify-center">
				<div class="w-1/5">{turn}</div>
				<div class="w-2/5 text-left">{currentPhase}</div>
				<div class="flex w-2/5 space-x-2">
					<button
						onclick={nextPhase}
						disabled={currentPhase !== PHASES[0] || players.length < 2}
						class="flex w-1/2 justify-center rounded bg-blue-500 px-4 py-2 text-white transition hover:bg-blue-600 disabled:bg-gray-50 disabled:text-gray-500"
					>
						{#if !isGameStarted}
							<PlayOutline />
						{:else}
							<CheckCircleOutline />
						{/if}
					</button>
					<button
						disabled={!isGameStarted || currentPhase === PHASES[0]}
						onclick={resumeGame}
						class="flex w-1/2 justify-center rounded bg-blue-500 px-4 py-2 text-white transition hover:bg-blue-600 disabled:bg-gray-50 disabled:text-gray-500"
					>
						{#if isGameActive}
							<PauseOutline />
						{:else}
							<PlayOutline />
						{/if}
					</button>
				</div>
			</div>
		</div>
		{#if !isGameStarted}
			<div class="shadow-mdp-1 w-full space-y-2 rounded-lg bg-white p-4 text-center">
				<label class="flex items-center justify-center text-sm font-medium text-gray-700">
					<span class="w-1/2 text-left"> Clock Type</span>
					<select bind:value={clocktype} class="w-1/2 rounded border border-gray-300 px-2 py-1">
						{#each CLOCK_TYPES as type}
							<option value={type}>{type.name}</option>
						{/each}
					</select>
				</label>
				<label class="flex items-center justify-center text-sm font-medium text-gray-700">
					<span class=" w-1/2 text-left"> Base Time (seconds)</span>
					<input
						type="number"
						bind:value={clocktype.baseTimeInS}
						class="w-1/2 rounded border border-gray-300 px-2 py-1"
					/></label
				>
				<label class="flex items-center justify-center text-sm font-medium text-gray-700">
					<span class=" w-1/2 text-left"> Increment Time (seconds)</span>
					<input
						type="number"
						bind:value={clocktype.incrementTimeInS}
						class="w-1/2 rounded border border-gray-300 px-2 py-1"
					/></label
				>
			</div>
		{/if}
		{#each players as player}
			<div
				class="w-full rounded-lg p-6 text-center shadow-md {player.isActive &&
				currentPhase !== PHASES[0]
					? 'bg-green-100 text-green-800 dark:bg-green-900 dark:text-green-300'
					: 'bg-white'} flex items-center justify-center"
			>
				<h2 class="flex w-2/5 items-center justify-center">
					<div class="w-1/6">
						{#if firstPlayer === player}
							<AwardOutline class="ml-2" />
						{/if}
					</div>
					<div class="w-1/6">
						<CloseCircleOutline class="ml-2" color={player.hasPassed ? 'red' : 'gray'} />
					</div>
					<div class="w-4/6 text-left">
						{#if !isGameStarted}
							<input
								type="text"
								bind:value={player.name}
								class="w-full rounded border border-gray-300 px-2 py-1"
							/>
						{:else}
							{player.name}
						{/if}
					</div>
				</h2>
				<div class="flex w-1/5 items-center justify-center">
					<ClockOutline />&nbsp;
					{Math.floor(player.time / 60)}:{String(player.time % 60).padStart(2, '0')}
				</div>

				<div class="flex w-2/5 space-x-2">
					{#if isGameStarted}
						<button
							disabled={!player.isActive ||
								!isGameActive ||
								player.hasPassed ||
								currentPhase !== PHASES[2]}
							onclick={() => {
								play(player);
							}}
							class="flex w-1/2 justify-center rounded bg-blue-500 px-4 py-2 text-white transition hover:bg-blue-600 disabled:bg-gray-50 disabled:text-gray-500"
						>
							{#if !isGameActive}
								<PauseOutline />
							{:else}
								<RocketOutline />
							{/if}
						</button>
						<button
							disabled={!player.isActive ||
								!isGameActive ||
								player.hasPassed ||
								currentPhase === PHASES[0]}
							onclick={() => {
								pass(player);
							}}
							class="flex w-1/2 justify-center rounded bg-orange-500 px-4 py-2 text-white transition hover:bg-orange-600 disabled:bg-gray-50 disabled:text-gray-500"
						>
							{#if !isGameActive}
								<PauseOutline />
							{:else}
								<CloseCircleOutline />
							{/if}
						</button>
					{:else}
						<button
							onclick={() => removePlayer(player)}
							class="flex w-full justify-center text-red-500"><UserRemoveOutline /></button
						>
					{/if}
				</div>
			</div>
		{/each}
		{#if !isGameStarted}
			<div
				class="flex w-full items-center justify-center rounded-lg bg-white p-6 text-center shadow-md"
			>
				<button
					disabled={players.length > 4}
					onclick={addPlayer}
					class="flex w-full justify-center text-green-500 disabled:bg-gray-50 disabled:text-gray-500"
					><UserAddOutline /></button
				>
			</div>
		{/if}
	</div>
</div>
