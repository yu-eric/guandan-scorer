<script lang="ts">
	import { onMount } from 'svelte';
	import { toPng } from 'html-to-image';

	type GameMode = '2v2' | '3v3';
	type Screen = 'mode-select' | 'team-setup' | 'game' | 'winner';
	type TeamId = 1 | 2;
	type TrumpTeam = TeamId | null;
	type Combo = { nameZh: string; nameEn: string; positions: string; points: number; description: string };
	type RoundHistory = {
		round: number;
		winner: string;
		combo: string;
		points: number;
		team1Level: number;
		team2Level: number;
		trumpTeam: TrumpTeam;
		aceChallengeTeam: TeamId | null;
		aceChallengeRemaining: number;
		team1Places: number[];
		team2Places: number[];
	};

	let currentScreen = $state<Screen>('mode-select');
	let gameMode = $state<GameMode>('2v2');
	let team1Level = $state(2);
	let team2Level = $state(2);
	let team1Name = $state('Team 1');
	let team2Name = $state('Team 2');
	let team1Players = $state<string[]>(['Player 1', 'Player 2']);
	let team2Players = $state<string[]>(['Player 3', 'Player 4']);
	let showScoring = $state(false);
	let roundHistory = $state<RoundHistory[]>([]);
	let roundNumber = $state(1);
	let isInitialSetup = $state(true);
	let gameWinner = $state<string>('');
	let gameDuration = $state(0);
	let showOptions = $state(false);
	let darkMode = $state(false);
	let language = $state<'en' | 'zh'>('en');
	let trumpTeam = $state<TrumpTeam>(null);
	let aceChallengeTeam = $state<TeamId | null>(null);
	let aceChallengeRemaining = $state(0);
	let roundPlacesTeam1 = $state<string[]>(['', '']);
	let roundPlacesTeam2 = $state<string[]>(['', '']);
	let graphEl = $state<HTMLDivElement | null>(null);
	let isExportingGraph = $state(false);
	let exportError = $state('');
	let showResetConfirm = $state(false);

	const levelCards = ['2', '3', '4', '5', '6', '7', '8', '9', '10', 'J', 'Q', 'K', 'A'];

	function getLevelCard(level: number): string {
		if (level < 2) return '2';
		if (level > 14) return 'A';
		return levelCards[level - 2];
	}

	// Scoring combos for 2v2 mode
	const combos2v2 = [
		{ nameZh: '双下', nameEn: 'Double Down', positions: '1-2', points: 3, description: 'Both finish 1st and 2nd' },
		{ nameZh: '单下', nameEn: 'Single Down', positions: '1-3', points: 2, description: 'One finishes 1st, other 3rd' },
		{ nameZh: '单下', nameEn: 'Single Down', positions: '1-4', points: 1, description: 'One finishes 1st, other 4th' }
	];

	// Scoring combos for 3v3 mode
	const combos3v3 = [
		{ nameZh: '三下', nameEn: 'Triple Down', positions: '1-2-3', points: 4, description: 'All three finish 1st, 2nd, 3rd' },
		{ nameZh: '两下半', nameEn: 'Two Down Half', positions: '1-2-4', points: 3, description: 'Two finish 1st and 2nd' },
		{ nameZh: '两下半', nameEn: 'Two Down Half', positions: '1-2-5', points: 3, description: 'Two finish 1st and 2nd' },
		{ nameZh: '两下半', nameEn: 'Two Down Half', positions: '1-2-6', points: 3, description: 'Two finish 1st and 2nd' },
		{ nameZh: '单下', nameEn: 'Single Down', positions: '1-3-4', points: 2, description: 'One finishes 1st' },
		{ nameZh: '单下', nameEn: 'Single Down', positions: '1-3-5', points: 2, description: 'One finishes 1st' },
		{ nameZh: '单下', nameEn: 'Single Down', positions: '1-3-6', points: 2, description: 'One finishes 1st' },
		{ nameZh: '单下', nameEn: 'Single Down', positions: '1-4-5', points: 2, description: 'One finishes 1st' },
		{ nameZh: '单下', nameEn: 'Single Down', positions: '1-4-6', points: 2, description: 'One finishes 1st' },
		{ nameZh: '单下', nameEn: 'Single Down', positions: '1-5-6', points: 2, description: 'One finishes 1st' },
		{ nameZh: '单上', nameEn: 'Single Up', positions: '2-3-4', points: 0, description: 'None finish 1st' },
		{ nameZh: '单上', nameEn: 'Single Up', positions: '2-3-5', points: 0, description: 'None finish 1st' },
		{ nameZh: '单上', nameEn: 'Single Up', positions: '2-3-6', points: 0, description: 'None finish 1st' },
		{ nameZh: '单上', nameEn: 'Single Up', positions: '2-4-5', points: 0, description: 'None finish 1st' },
		{ nameZh: '单上', nameEn: 'Single Up', positions: '2-4-6', points: 0, description: 'None finish 1st' },
		{ nameZh: '单上', nameEn: 'Single Up', positions: '2-5-6', points: 0, description: 'None finish 1st' }
	];

	function getComboName(combo: { nameZh: string; nameEn: string }) {
		return language === 'en' ? combo.nameEn : combo.nameZh;
	}

	function getCurrentCombos() {
		return gameMode === '2v2' ? combos2v2 : combos3v3;
	}

	function getTrumpLevel(): number {
		if (trumpTeam === null) return 0;
		return trumpTeam === 1 ? team1Level : team2Level;
	}

	function getTrumpTeamName(): string {
		if (trumpTeam === null) return language === 'en' ? 'None' : '无';
		return trumpTeam === 1 ? team1Name : team2Name;
	}

	function getEffectivePoints(combo: { points: number }, winnerTeam: TeamId): number {
		// Round 1: no trump, no switch-cost.
		if (trumpTeam === null) return combo.points;
		// If the non-trump side wins, spend 1 point to switch trump before adding.
		const switching = winnerTeam !== trumpTeam;
		return switching ? Math.max(combo.points - 1, 0) : combo.points;
	}

	function isQualifyingAceWin(combo: { positions: string }): boolean {
		const positions = combo.positions.split('-').map(Number);
		return (positions.includes(1) && positions.includes(2)) || (positions.includes(1) && positions.includes(3));
	}

	function resetRoundPlaces() {
		roundPlacesTeam1 = Array(team1Players.length).fill('');
		roundPlacesTeam2 = Array(team2Players.length).fill('');
	}

	function openScoring() {
		resetRoundPlaces();
		showScoring = !showScoring;
	}

	function getRoundPlacesCount(): number {
		return gameMode === '2v2' ? 4 : 6;
	}

	function getChosenPlaces(): Set<string> {
		const chosen = new Set<string>();
		for (const v of roundPlacesTeam1) if (v) chosen.add(v);
		for (const v of roundPlacesTeam2) if (v) chosen.add(v);
		return chosen;
	}

	function isRoundPlacesCompleteAndUnique(): boolean {
		const totalPlayers = team1Players.length + team2Players.length;
		const all = [...roundPlacesTeam1, ...roundPlacesTeam2];
		if (all.some((v) => !v)) return false;
		const chosen = new Set(all);
		return chosen.size === totalPlayers;
	}

	function parseRoundPlaces(): { team1: number[]; team2: number[] } | null {
		if (!isRoundPlacesCompleteAndUnique()) return null;
		const team1 = roundPlacesTeam1.map((v) => Number(v));
		const team2 = roundPlacesTeam2.map((v) => Number(v));
		return { team1, team2 };
	}

	function deriveWinnerAndComboFromPlaces(places: { team1: number[]; team2: number[] }): { winnerTeam: TeamId; combo: Combo } | null {
		const team1HasFirst = places.team1.includes(1);
		const team2HasFirst = places.team2.includes(1);
		if (team1HasFirst === team2HasFirst) return null;

		const winnerTeam: TeamId = team1HasFirst ? 1 : 2;
		const winnerPlaces = (winnerTeam === 1 ? places.team1 : places.team2).slice().sort((a, b) => a - b);
		const positions = winnerPlaces.join('-');
		const combo = getCurrentCombos().find((c) => c.positions === positions);
		if (!combo) return null;
		return { winnerTeam, combo };
	}

	function selectMode(mode: GameMode) {
		gameMode = mode;
		// Reset player arrays based on mode
		if (mode === '2v2') {
			team1Players = ['Player 1', 'Player 2'];
			team2Players = ['Player 3', 'Player 4'];
			roundPlacesTeam1 = ['', ''];
			roundPlacesTeam2 = ['', ''];
		} else {
			team1Players = ['Player 1', 'Player 2', 'Player 3'];
			team2Players = ['Player 4', 'Player 5', 'Player 6'];
			roundPlacesTeam1 = ['', '', ''];
			roundPlacesTeam2 = ['', '', ''];
		}
		currentScreen = 'team-setup';
	}

	function startGame() {
		currentScreen = 'game';
		isInitialSetup = false;
	}

	function backToSetup() {
		currentScreen = 'team-setup';
		showScoring = false;
	}

	function backToMode() {
		currentScreen = 'mode-select';
	}

	function scoreRoundFromPlaces() {
		const places = parseRoundPlaces();
		if (!places) return;
		const derived = deriveWinnerAndComboFromPlaces(places);
		if (!derived) return;

		const { winnerTeam, combo } = derived;
		const winnerName = winnerTeam === 1 ? team1Name : team2Name;
		const points = getEffectivePoints(combo, winnerTeam);
		const comboName = getComboName(combo);
		const challengeWasActiveAtStart = aceChallengeTeam !== null;
		const winnerLevelBefore = winnerTeam === 1 ? team1Level : team2Level;

		// If a team is already on Ace, they must win a qualifying combo to win the match.
		const wonGame = winnerLevelBefore === 14 && isQualifyingAceWin(combo);

		// Add to history (store pre-round state for undo)
		roundHistory.push({
			round: roundNumber,
			winner: winnerName,
			combo: `${comboName} (${combo.positions})`,
			points: points,
			team1Level: team1Level,
			team2Level: team2Level,
			trumpTeam: trumpTeam,
			aceChallengeTeam: aceChallengeTeam,
			aceChallengeRemaining: aceChallengeRemaining,
			team1Places: places.team1,
			team2Places: places.team2
		});

		// If won, show winner screen
		if (wonGame) {
			gameWinner = winnerName;
			gameDuration = roundNumber;
			currentScreen = 'winner';
			showScoring = false;
			return;
		}

		// Round 1: establish trump, but don't treat it as a "switch".
		if (trumpTeam === null) {
			trumpTeam = winnerTeam;
		} else if (winnerTeam !== trumpTeam) {
			// If the non-trump side wins, switch trump.
			trumpTeam = winnerTeam;
		}

		// Update levels (cap at Ace)
		if (winnerTeam === 1) {
			team1Level = Math.min(team1Level + points, 14);
		} else {
			team2Level = Math.min(team2Level + points, 14);
		}

		// If a team just reached Ace this round, start the 3-game countdown.
		const winnerLevelAfter = winnerTeam === 1 ? team1Level : team2Level;
		if (winnerLevelBefore < 14 && winnerLevelAfter === 14) {
			aceChallengeTeam = winnerTeam;
			aceChallengeRemaining = 3;
		}

		// If the Ace challenge was already active at the start of this round, decrement.
		if (challengeWasActiveAtStart && aceChallengeTeam !== null) {
			aceChallengeRemaining = Math.max(aceChallengeRemaining - 1, 0);
			if (aceChallengeRemaining === 0) {
				if (aceChallengeTeam === 1) team1Level = 2;
				if (aceChallengeTeam === 2) team2Level = 2;
				aceChallengeTeam = null;
				aceChallengeRemaining = 0;
			}
		}

		// Reset and close
		roundNumber++;
		showScoring = false;
	}

	function cancelScoring() {
		showScoring = false;
	}

	function resetGame() {
		team1Level = 2;
		team2Level = 2;
		team1Name = 'Team 1';
		team2Name = 'Team 2';
		team1Players = ['Player 1', 'Player 2'];
		team2Players = ['Player 3', 'Player 4'];
		roundHistory = [];
		roundNumber = 1;
		currentScreen = 'mode-select';
		showScoring = false;
		isInitialSetup = true;
		gameWinner = '';
		gameDuration = 0;
		trumpTeam = null;
		aceChallengeTeam = null;
		aceChallengeRemaining = 0;
		roundPlacesTeam1 = ['', ''];
		roundPlacesTeam2 = ['', ''];
		graphEl = null;
		isExportingGraph = false;
		exportError = '';
		showResetConfirm = false;
	}

	function requestResetGame() {
		showResetConfirm = true;
	}

	function confirmResetGame() {
		resetGame();
	}

	function undoLastRound() {
		if (roundHistory.length === 0) return;
		
		const lastRound = roundHistory[roundHistory.length - 1];
		team1Level = lastRound.team1Level;
		team2Level = lastRound.team2Level;
		trumpTeam = lastRound.trumpTeam;
		aceChallengeTeam = lastRound.aceChallengeTeam;
		aceChallengeRemaining = lastRound.aceChallengeRemaining;
		roundHistory.pop();
		roundNumber--;
	}

	const playerColors = [
		'#f5576c',
		'#4facfe',
		'#fcb69f',
		'#00c2a8',
		'#ffd166',
		'#a78bfa'
	];

	function getAllPlayers(): Array<{ key: string; name: string; team: TeamId; color: string }> {
		const team1 = team1Players.map((name, index) => ({ key: `t1-${index}`, name, team: 1 as const }));
		const team2 = team2Players.map((name, index) => ({ key: `t2-${index}`, name, team: 2 as const }));
		return [...team1, ...team2].map((p, i) => ({ ...p, color: playerColors[i % playerColors.length] }));
	}

	function getPlayerPlacesForRound(round: RoundHistory, playerKey: string): number | null {
		const [teamPart, idxPart] = playerKey.split('-');
		const index = Number(idxPart);
		if (teamPart === 't1') return round.team1Places?.[index] ?? null;
		if (teamPart === 't2') return round.team2Places?.[index] ?? null;
		return null;
	}

	function getPlayerPlacesOverGame(playerKey: string): number[] {
		return roundHistory
			.map((r) => getPlayerPlacesForRound(r, playerKey))
			.filter((v): v is number => typeof v === 'number');
	}

	function getPlayerAveragePlace(playerKey: string): number {
		const values = getPlayerPlacesOverGame(playerKey);
		if (values.length === 0) return NaN;
		return values.reduce((sum, v) => sum + v, 0) / values.length;
	}

	function getPlayerAverageRows(): Array<{ key: string; name: string; team: TeamId; avg: number }> {
		return getAllPlayers()
			.map((p) => ({ key: p.key, name: p.name, team: p.team, avg: getPlayerAveragePlace(p.key) }))
			.filter((p) => Number.isFinite(p.avg))
			.sort((a, b) => a.avg - b.avg);
	}

	function isProbablyMobile(): boolean {
		const nav = navigator as unknown as { userAgentData?: { mobile?: boolean } };
		if (typeof nav.userAgentData?.mobile === 'boolean') return !!nav.userAgentData.mobile;
		return /Android|iPhone|iPad|iPod/i.test(navigator.userAgent);
	}

	async function shareOrDownloadGraphImage() {
		exportError = '';
		if (!graphEl) return;
		try {
			isExportingGraph = true;
			const dataUrl = await toPng(graphEl, {
				cacheBust: true,
				pixelRatio: 2,
				backgroundColor: '#ffffff'
			});
			const res = await fetch(dataUrl);
			const blob = await res.blob();
			const file = new File([blob], `guandan-placements-${new Date().toISOString().slice(0, 10)}.png`, {
				type: 'image/png'
			});

			const nav = navigator as unknown as {
				share?: (data: { files?: File[]; title?: string; text?: string }) => Promise<void>;
				canShare?: (data: { files?: File[] }) => boolean;
			};
			const win = window as unknown as {
				showSaveFilePicker?: (options?: {
					suggestedName?: string;
					types?: Array<{ description?: string; accept: Record<string, string[]> }>;
				}) => Promise<{ createWritable: () => Promise<{ write: (data: Blob) => Promise<void>; close: () => Promise<void> }> }>;
			};

			// Mobile: prefer the native share sheet.
			if (isProbablyMobile() && nav.share && nav.canShare?.({ files: [file] })) {
				await nav.share({
					title: 'Guandan placements',
					text: 'Player placements over rounds',
					files: [file]
				});
				return;
			}

			// Desktop: prefer native save dialog when available (Chrome/Edge File System Access API).
			if (win.showSaveFilePicker) {
				const handle = await win.showSaveFilePicker({
					suggestedName: file.name,
					types: [{ description: 'PNG Image', accept: { 'image/png': ['.png'] } }]
				});
				const writable = await handle.createWritable();
				await writable.write(blob);
				await writable.close();
				return;
			}

			// Otherwise, if share is available (some desktops), use it.
			if (nav.share && nav.canShare?.({ files: [file] })) {
				await nav.share({
					title: 'Guandan placements',
					text: 'Player placements over rounds',
					files: [file]
				});
				return;
			}

			const a = document.createElement('a');
			a.href = dataUrl;
			a.download = file.name;
			a.click();
		} catch (e) {
			exportError = e instanceof Error ? e.message : 'Failed to export image';
		} finally {
			isExportingGraph = false;
		}
	}
</script>

<div class="container" class:dark-mode={darkMode}>
	{#if currentScreen === 'mode-select'}
		<!-- Mode Selection Screen -->
		<div class="screen-content mode-select-screen">
			<div class="mode-header">
				<h1>🎴 Guandan Scorer</h1>
			</div>
			<div class="mode-cards">
				<button class="mode-card" onclick={() => selectMode('2v2')}>
					<div class="mode-icon">👥</div>
					<h2>2v2</h2>
					<p>Two teams of 2 players each</p>
				</button>
				<button class="mode-card" onclick={() => selectMode('3v3')}>
					<div class="mode-icon">👥👥👥</div>
					<h2>3v3</h2>
					<p>Two teams of 3 players each</p>
				</button>
			</div>
		</div>
	{:else if currentScreen === 'team-setup'}
		<!-- Team Setup Screen -->
		<div class="screen-content setup-screen">
			<h1>🎴 Setup Teams</h1>
			<p class="subtitle">Click on any name to edit</p>
			
			<div class="setup-teams">
				<!-- Team 1 -->
				<div class="setup-team team1">
					<input 
						type="text" 
						class="team-name-input"
						bind:value={team1Name}
						placeholder="Team 1 Name"
					/>
					<div class="setup-players">
						<input 
							type="text" 
							bind:value={team1Players[0]}
							placeholder="Player 1"
						/>
						<input 
							type="text" 
							bind:value={team1Players[1]}
							placeholder="Player 2"
						/>
						{#if gameMode === '3v3'}
							<input 
								type="text" 
								bind:value={team1Players[2]}
								placeholder="Player 3"
							/>
						{/if}
					</div>
				</div>

				<!-- Team 2 -->
				<div class="setup-team team2">
					<input 
						type="text" 
						class="team-name-input"
						bind:value={team2Name}
						placeholder="Team 2 Name"
					/>
					<div class="setup-players">
						<input 
							type="text" 
							bind:value={team2Players[0]}
							placeholder={gameMode === '2v2' ? 'Player 3' : 'Player 4'}
						/>
						<input 
							type="text" 
							bind:value={team2Players[1]}
							placeholder={gameMode === '2v2' ? 'Player 4' : 'Player 5'}
						/>
						{#if gameMode === '3v3'}
							<input 
								type="text" 
								bind:value={team2Players[2]}
								placeholder="Player 6"
							/>
						{/if}
					</div>
				</div>
			</div>

			<div class="setup-actions">
				<button class="start-btn" onclick={startGame}>{isInitialSetup ? 'Start Game' : 'Save'}</button>
			</div>
		</div>
	{:else if currentScreen === 'winner'}
		<!-- Winner Screen -->
			<div class="screen-content winner-screen">
			<div class="winner-card">
				<div class="trophy">🏆</div>
				<h1>Game Over!</h1>
				<h2>{gameWinner} Wins!</h2>
				<div class="winner-stats">
					<div class="stat">
						<span class="stat-label">Total Rounds</span>
						<span class="stat-value">{gameDuration}</span>
					</div>
					<div class="stat">
						<span class="stat-label">Final Scores</span>
						<span class="stat-value">{getLevelCard(team1Level)} - {getLevelCard(team2Level)}</span>
					</div>
					<div class="stat">
						<span class="stat-label">Game Mode</span>
						<span class="stat-value">{gameMode}</span>
					</div>
				</div>
				
				{#if roundHistory.length > 0}
					<div class="game-summary">
						<h3>Game Summary</h3>
						<div class="summary-table">
							<table>
								<thead>
									<tr>
										<th>Round</th>
										<th>Winner</th>
										<th>Combo</th>
										<th>Points</th>
									</tr>
								</thead>
								<tbody>
									{#each roundHistory as round}
										<tr>
											<td>{round.round}</td>
											<td>{round.winner}</td>
											<td>{round.combo}</td>
											<td>+{round.points}</td>
										</tr>
									{/each}
								</tbody>
							</table>
						</div>
					</div>

					<div class="game-summary">
						<h3>Player Placements</h3>
						<div class="placements-actions">
							<button class="share-btn" onclick={shareOrDownloadGraphImage} disabled={isExportingGraph}>
								{isExportingGraph ? 'Preparing…' : 'Share / Download Graph'}
							</button>
							{#if exportError}
								<div class="export-error">{exportError}</div>
							{/if}
						</div>
						<div class="placements-graph" bind:this={graphEl}>
							<svg viewBox="0 0 760 320" role="img" aria-label="Player placements by round">
								{#if roundHistory.length === 1}
									<text x="380" y="160" text-anchor="middle" fill="#666" font-size="16">Add more rounds to see trends</text>
								{:else}
									{@const rounds = roundHistory.length}
									{@const placesCount = getRoundPlacesCount()}
									{@const padL = 60}
									{@const padR = 20}
									{@const padT = 20}
									{@const padB = 40}
									{@const w = 760}
									{@const h = 320}
									{@const plotW = w - padL - padR}
									{@const plotH = h - padT - padB}
									<rect x={padL} y={padT} width={plotW} height={plotH} fill="#fff" opacity="0.0" />
									{#each Array(placesCount) as _, i}
										{@const place = i + 1}
										{@const y = padT + (place - 1) * (plotH / (placesCount - 1))}
										<line x1={padL} y1={y} x2={w - padR} y2={y} stroke="rgba(0,0,0,0.08)" />
										<text x={padL - 10} y={y + 4} text-anchor="end" fill="#666" font-size="12">{place}</text>
									{/each}

									{#each getAllPlayers() as player}
										{@const values = getPlayerPlacesOverGame(player.key)}
										{@const points = values
											.map((place, idx) => {
												const x = padL + (idx * plotW) / (rounds - 1);
												const y = padT + (place - 1) * (plotH / (placesCount - 1));
												return `${x},${y}`;
											})
											.join(' ')}
										<polyline points={points} fill="none" stroke={player.color} stroke-width="3" stroke-linecap="round" stroke-linejoin="round" opacity="0.9">
											<title>{player.name}</title>
										</polyline>
									{/each}

									{#each Array(rounds) as _, i}
										{@const x = padL + (i * plotW) / (rounds - 1)}
										<text x={x} y={h - 14} text-anchor="middle" fill="#666" font-size="12">{i + 1}</text>
									{/each}
								{/if}
							</svg>
							<div class="placements-legend">
								{#each getAllPlayers() as player}
									<div class="legend-item">
										<svg class="legend-line" viewBox="0 0 40 8" aria-hidden="true">
											<line
												x1="2"
												y1="4"
												x2="38"
												y2="4"
												stroke={player.color}
												stroke-width="3"
												stroke-linecap="round"
											/>
										</svg>
										<span class="legend-name">{player.name}</span>
									</div>
								{/each}
							</div>
						</div>

						<div class="summary-table">
							<table>
								<thead>
									<tr>
										<th>Player</th>
										<th>Team</th>
										<th>Avg Place</th>
									</tr>
								</thead>
								<tbody>
									{#each getPlayerAverageRows() as row}
										<tr>
											<td>{row.name}</td>
											<td>{row.team === 1 ? team1Name : team2Name}</td>
											<td>{row.avg.toFixed(2)}</td>
										</tr>
									{/each}
								</tbody>
							</table>
						</div>
					</div>
				{/if}

				<button class="new-game-btn" onclick={requestResetGame}>🎴 New Game</button>
			</div>
		</div>
	{:else}
		<!-- Game Screen -->
		<div class="screen-content game-screen">
			<header>
				<div class="title-area">
					<h1>🎴 Guandan Scorer</h1>
					<div class="trump-badge">
						<span class="trump-label">Trump</span>
						<span class="trump-team">{getTrumpTeamName()}</span>
						{#if trumpTeam !== null}
							<span class="trump-card">{getLevelCard(getTrumpLevel())}</span>
							<span class="trump-level">(Lvl {getTrumpLevel()})</span>
						{:else}
							<span class="trump-level">(Round 1)</span>
						{/if}
					</div>
				</div>
				<div class="header-buttons">
					<button class="options-btn" onclick={() => (showOptions = !showOptions)}>⚙️</button>
					<button class="edit-btn" onclick={backToSetup}>✏️ Edit Teams</button>
				</div>
			</header>

			<div class="teams-container">
				<!-- Team 1 -->
				<div class="team-card team1" class:trump-team={trumpTeam === 1}>
					<div class="team-header">
						<h2>{team1Name}</h2>
						<div class="level-display">
							<span class="level-label">Level</span>
							<span class="level-card">{getLevelCard(team1Level)}</span>
							<span class="level-number">(Lvl {team1Level})</span>
						</div>
					</div>
					<div class="players">
						{#each team1Players as player}
							<div class="player">
								<span>👤 {player}</span>
							</div>
						{/each}
					</div>
				</div>

				<!-- Team 2 -->
				<div class="team-card team2" class:trump-team={trumpTeam === 2}>
					<div class="team-header">
						<h2>{team2Name}</h2>
						<div class="level-display">
							<span class="level-label">Level</span>
							<span class="level-card">{getLevelCard(team2Level)}</span>
							<span class="level-number">(Lvl {team2Level})</span>
						</div>
					</div>
					<div class="players">
						{#each team2Players as player}
							<div class="player">
								<span>👤 {player}</span>
							</div>
						{/each}
					</div>
				</div>


			</div>

			<div class="actions">
				<button class="score-btn" onclick={openScoring}>
					📊 Score Round
				</button>
				<button class="reset-btn" onclick={requestResetGame}>🔄 Reset Game</button>
			</div>

			<!-- Round History Panel -->
			{#if roundHistory.length > 0}
				<div class="history-panel">
					<div class="history-header">
						<h3>Round History</h3>
						<button class="undo-btn" onclick={undoLastRound}>↶ Undo Last</button>
					</div>
					<div class="history-table">
						<table>
							<thead>
								<tr>
									<th>Round</th>
									<th>Winner</th>
									<th>Combo</th>
									<th>Points</th>
									<th>Levels After</th>
								</tr>
							</thead>
							<tbody>
								{#each roundHistory as round}
									<tr>
										<td>{round.round}</td>
										<td>{round.winner}</td>
										<td>{round.combo}</td>
										<td>+{round.points}</td>
										<td>{getLevelCard(round.team1Level)} / {getLevelCard(round.team2Level)}</td>
									</tr>
								{/each}
							</tbody>
						</table>
					</div>
				</div>
			{/if}

			<!-- Scoring Modal -->
			{#if showScoring}
				<div class="info-modal-overlay" onclick={() => (showScoring = false)} onkeydown={(e) => e.key === 'Escape' && (showScoring = false)} role="button" tabindex="0">
					<div class="info-modal scoring-modal" onclick={(e) => e.stopPropagation()} onkeydown={(e) => e.stopPropagation()} role="dialog" tabindex="-1">
						<div class="info-header">
							<h3>Enter Round Placements</h3>
							<button class="close-btn" onclick={() => (showScoring = false)}>✕</button>
						</div>
						<div class="info-content scoring-content">
							<div class="placements-grid">
								<div class="placements-team">
									<h4>{team1Name}</h4>
									{#each team1Players as player, i}
										<label class="placement-row">
											<span class="placement-player">{player}</span>
											<select class="placement-select" bind:value={roundPlacesTeam1[i]}>
												<option value="">--</option>
												{#each Array(getRoundPlacesCount()) as _, p}
													{@const value = String(p + 1)}
													{@const chosen = getChosenPlaces()}
													<option value={value} disabled={chosen.has(value) && roundPlacesTeam1[i] !== value}>{p + 1}</option>
												{/each}
											</select>
										</label>
									{/each}
								</div>

								<div class="placements-team">
									<h4>{team2Name}</h4>
									{#each team2Players as player, i}
										<label class="placement-row">
											<span class="placement-player">{player}</span>
											<select class="placement-select" bind:value={roundPlacesTeam2[i]}>
												<option value="">--</option>
												{#each Array(getRoundPlacesCount()) as _, p}
													{@const value = String(p + 1)}
													{@const chosen = getChosenPlaces()}
													<option value={value} disabled={chosen.has(value) && roundPlacesTeam2[i] !== value}>{p + 1}</option>
												{/each}
											</select>
										</label>
									{/each}
								</div>
							</div>

							<div class="scoring-actions">
								<button class="start-btn" disabled={!isRoundPlacesCompleteAndUnique()} onclick={scoreRoundFromPlaces}>
									Score Round
								</button>
								<button class="cancel-btn" onclick={cancelScoring}>Cancel</button>
							</div>
						</div>
					</div>
				</div>
			{/if}

			<!-- Options Modal -->
			{#if showOptions}
				<div class="info-modal-overlay" onclick={() => (showOptions = false)} onkeydown={(e) => e.key === 'Escape' && (showOptions = false)} role="button" tabindex="0">
					<div class="info-modal options-modal" onclick={(e) => e.stopPropagation()} onkeydown={(e) => e.stopPropagation()} role="dialog" tabindex="-1">
						<div class="info-header">
							<h3>⚙️ Options</h3>
							<button class="close-btn" onclick={() => (showOptions = false)}>✕</button>
						</div>
						<div class="info-content options-content">
							<div class="option-group">
								<label class="option-label">
									<span class="option-text">Theme</span>
									<div class="toggle-group">
										<button 
											class="toggle-btn {darkMode ? '' : 'active'}"
											onclick={() => (darkMode = false)}
										>☀️ Light</button>
										<button 
											class="toggle-btn {darkMode ? 'active' : ''}"
											onclick={() => (darkMode = true)}
										>🌙 Dark</button>
									</div>
								</label>
							</div>
							<div class="option-group">
								<label class="option-label">
									<span class="option-text">Language</span>
									<div class="toggle-group">
										<button 
											class="toggle-btn {language === 'en' ? 'active' : ''}"
											onclick={() => (language = 'en')}
										>🇬🇧 English</button>
										<button 
											class="toggle-btn {language === 'zh' ? 'active' : ''}"
											onclick={() => (language = 'zh')}
										>🇨🇳 中文</button>
									</div>
								</label>
							</div>
						</div>
					</div>
				</div>
			{/if}

			<!-- Reset Confirm Modal -->
			{#if showResetConfirm}
				<div class="info-modal-overlay" onclick={() => (showResetConfirm = false)} onkeydown={(e) => e.key === 'Escape' && (showResetConfirm = false)} role="button" tabindex="0">
					<div class="info-modal options-modal" onclick={(e) => e.stopPropagation()} onkeydown={(e) => e.stopPropagation()} role="dialog" tabindex="-1">
						<div class="info-header">
							<h3>Confirm Reset</h3>
							<button class="close-btn" onclick={() => (showResetConfirm = false)}>✕</button>
						</div>
						<div class="info-content options-content">
							<p class="confirm-text">This will reset scores, rounds, and history. Continue?</p>
							<div class="confirm-actions">
								<button class="cancel-btn" onclick={() => (showResetConfirm = false)}>Cancel</button>
								<button class="start-btn" onclick={confirmResetGame}>Reset</button>
							</div>
						</div>
					</div>
				</div>
			{/if}
		</div>
	{/if}
</div>

<svelte:head>
	{#if darkMode}
		<style>
			body { background: linear-gradient(135deg, #1a1a2e 0%, #16213e 100%) !important; }
		</style>
	{/if}
</svelte:head>

<style>
	:global(body) {
		margin: 0;
		padding: 0;
		font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu,
			Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
		background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
		min-height: 100vh;
		transition: background 0.3s ease;
	}

	.container {
		max-width: 1200px;
		margin: 0 auto;
		padding: 2rem;
		min-height: 100vh;
	}

	.screen-content {
		animation: fadeIn 0.4s ease;
	}

	@keyframes fadeIn {
		from {
			opacity: 0;
			transform: translateY(20px);
		}
		to {
			opacity: 1;
			transform: translateY(0);
		}
	}

	/* Mode Selection Screen */
	.mode-select-screen {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		min-height: 80vh;
	}

	.mode-header {
		position: relative;
		width: 100%;
		max-width: 700px;
		margin-bottom: 3rem;
	}

	.mode-header h1 {
		margin: 0;
		text-align: center;
	}

	.mode-select-screen h1 {
		color: white;
		font-size: 3.5rem;
		margin: 0;
		text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.3);
		text-align: center;
	}

	.mode-cards {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
		gap: 2rem;
		width: 100%;
		max-width: 700px;
	}

	.mode-card {
		background: white;
		border: none;
		border-radius: 30px;
		padding: 3rem 2rem;
		cursor: pointer;
		transition: all 0.3s ease;
		box-shadow: 0 10px 40px rgba(0, 0, 0, 0.2);
		text-align: center;
	}

	.mode-card:hover {
		transform: translateY(-10px) scale(1.02);
		box-shadow: 0 15px 50px rgba(0, 0, 0, 0.3);
	}

	.mode-icon {
		font-size: 4rem;
		margin-bottom: 1rem;
	}

	.mode-card h2 {
		font-size: 2.5rem;
		margin: 0 0 0.5rem 0;
		color: #667eea;
	}

	.mode-card p {
		font-size: 1.1rem;
		color: #666;
		margin: 0;
	}

	/* Winner Screen */
	.winner-screen {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		min-height: 80vh;
	}

	.winner-card {
		background: white;
		border-radius: 30px;
		padding: 3rem;
		box-shadow: 0 15px 60px rgba(0, 0, 0, 0.3);
		text-align: center;
		max-width: 800px;
		width: 100%;
		animation: scaleIn 0.5s ease;
	}

	@keyframes scaleIn {
		from {
			opacity: 0;
			transform: scale(0.8);
		}
		to {
			opacity: 1;
			transform: scale(1);
		}
	}

	.trophy {
		font-size: 6rem;
		margin-bottom: 1rem;
		animation: bounce 1s ease infinite;
	}

	@keyframes bounce {
		0%, 100% {
			transform: translateY(0);
		}
		50% {
			transform: translateY(-20px);
		}
	}

	.winner-card h1 {
		font-size: 3rem;
		margin: 0 0 1rem 0;
		color: #333;
	}

	.winner-card h2 {
		font-size: 2.5rem;
		margin: 0 0 2rem 0;
		background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
		-webkit-background-clip: text;
		-webkit-text-fill-color: transparent;
		background-clip: text;
	}

	.winner-stats {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
		gap: 1.5rem;
		margin-bottom: 2rem;
		padding: 1.5rem;
		background: linear-gradient(135deg, rgba(102, 126, 234, 0.1) 0%, rgba(118, 75, 162, 0.1) 100%);
		border-radius: 20px;
	}

	.stat {
		display: flex;
		flex-direction: column;
		gap: 0.5rem;
	}

	.stat-label {
		font-size: 0.9rem;
		color: #666;
		text-transform: uppercase;
		letter-spacing: 1px;
	}

	.stat-value {
		font-size: 1.8rem;
		font-weight: bold;
		color: #667eea;
	}

	.game-summary {
		margin-top: 2rem;
		text-align: left;
	}

	.game-summary h3 {
		font-size: 1.5rem;
		color: #333;
		margin-bottom: 1rem;
		text-align: center;
	}

	.summary-table {
		max-height: 300px;
		overflow-y: auto;
		border-radius: 15px;
		box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
	}

	.summary-table table {
		width: 100%;
		border-collapse: separate;
		border-spacing: 0;
	}

	.summary-table thead {
		background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
		color: white;
		position: sticky;
		top: 0;
	}

	.summary-table th {
		padding: 0.75rem;
		text-align: center;
		font-weight: bold;
		font-size: 0.9rem;
	}

	.summary-table td {
		padding: 0.75rem;
		text-align: center;
		border-bottom: 1px solid #e9ecef;
		font-size: 0.9rem;
	}

	.summary-table tbody tr {
		background: white;
	}

	.summary-table tbody tr:last-child td {
		border-bottom: none;
	}

	.summary-table tbody tr:hover {
		background: rgba(102, 126, 234, 0.05);
	}

	.new-game-btn {
		margin-top: 2rem;
		padding: 1.25rem 3rem;
		font-size: 1.3rem;
		font-weight: bold;
		border: none;
		border-radius: 50px;
		cursor: pointer;
		transition: all 0.3s ease;
		box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
		background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
		color: white;
	}

	.new-game-btn:hover {
		transform: translateY(-3px);
		box-shadow: 0 6px 20px rgba(0, 0, 0, 0.3);
	}


	/* Team Setup Screen */
	.setup-screen h1 {
		color: white;
		font-size: 3rem;
		margin: 0 0 0.5rem 0;
		text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.3);
		text-align: center;
	}

	.subtitle {
		color: rgba(255, 255, 255, 0.9);
		text-align: center;
		margin: 0 0 2rem 0;
		font-size: 1.1rem;
	}

	.setup-teams {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
		gap: 1.5rem;
		margin-bottom: 2rem;
	}

	.setup-team {
		background: white;
		border-radius: 20px;
		padding: 2rem;
		box-shadow: 0 8px 30px rgba(0, 0, 0, 0.2);
	}

	.setup-team.team1 {
		border-top: 5px solid #f093fb;
	}

	.setup-team.team2 {
		border-top: 5px solid #4facfe;
	}

	.team-name-input {
		width: 100%;
		padding: 1rem;
		font-size: 1.5rem;
		font-weight: bold;
		border: 3px solid #e9ecef;
		border-radius: 15px;
		margin-bottom: 1.5rem;
		outline: none;
		text-align: center;
		transition: all 0.3s ease;
		box-sizing: border-box;
	}

	.team-name-input:focus {
		border-color: #667eea;
		box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
	}

	.setup-players {
		display: flex;
		flex-direction: column;
		gap: 1rem;
	}

	.setup-players input {
		width: 100%;
		padding: 1rem;
		font-size: 1.1rem;
		border: 2px solid #e9ecef;
		border-radius: 10px;
		outline: none;
		transition: all 0.3s ease;
		box-sizing: border-box;
	}

	.setup-players input:focus {
		border-color: #667eea;
		box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
	}

	.setup-actions {
		display: flex;
		gap: 1rem;
		justify-content: center;
		margin-top: 2rem;
	}

	.start-btn {
		padding: 1.25rem 3rem;
		font-size: 1.3rem;
		font-weight: bold;
		border: none;
		border-radius: 50px;
		cursor: pointer;
		transition: all 0.3s ease;
		box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
		background: linear-gradient(135deg, #f093fb 0%, #f5576c 100%);
		color: white;
	}

	.start-btn:hover {
		transform: translateY(-3px);
		box-shadow: 0 6px 20px rgba(0, 0, 0, 0.3);
	}

	/* Game Screen */
	.game-screen header {
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		margin-bottom: 2rem;
		flex-wrap: wrap;
		gap: 1rem;
	}

	.game-screen h1 {
		color: white;
		font-size: 2.5rem;
		margin: 0;
		text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.3);
	}

	.title-area {
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 0.5rem;
	}

	.trump-badge {
		display: inline-flex;
		align-items: center;
		gap: 0.6rem;
		padding: 0.5rem 0.9rem;
		border-radius: 999px;
		background: rgba(255, 255, 255, 0.2);
		border: 2px solid rgba(255, 255, 255, 0.6);
		backdrop-filter: blur(10px);
		width: fit-content;
		justify-content: center;
	}

	.trump-label {
		font-weight: 700;
		color: rgba(255, 255, 255, 0.9);
		text-transform: uppercase;
		letter-spacing: 1px;
		font-size: 0.8rem;
	}

	.trump-team {
		color: white;
		font-weight: 700;
		font-size: 1.1rem;
	}

	.trump-card {
		background: rgba(255, 255, 255, 0.25);
		border: 1px solid rgba(255, 255, 255, 0.35);
		padding: 0.2rem 0.6rem;
		border-radius: 999px;
		color: white;
		font-weight: 800;
		font-size: 1.6rem;
		min-width: 56px;
		text-align: center;
	}

	.trump-level {
		color: rgba(255, 255, 255, 0.9);
		font-weight: 600;
		font-size: 0.9rem;
	}

	.header-buttons {
		display: flex;
		gap: 0.5rem;
		align-items: center;
		justify-content: center;
	}

	.options-btn {
		padding: 0.75rem 1rem;
		font-size: 1.2rem;
		border: 2px solid white;
		background: rgba(255, 255, 255, 0.2);
		color: white;
		border-radius: 50%;
		cursor: pointer;
		transition: all 0.3s ease;
		backdrop-filter: blur(10px);
		width: 48px;
		height: 48px;
		display: flex;
		align-items: center;
		justify-content: center;
	}

	.options-btn:hover {
		background: rgba(255, 255, 255, 0.3);
		transform: rotate(90deg);
	}

	.edit-btn {
		padding: 0.75rem 1.5rem;
		font-size: 1rem;
		font-weight: bold;
		border: 2px solid white;
		background: rgba(255, 255, 255, 0.2);
		color: white;
		border-radius: 50px;
		cursor: pointer;
		transition: all 0.3s ease;
		backdrop-filter: blur(10px);
	}

	.edit-btn:hover {
		background: rgba(255, 255, 255, 0.3);
		transform: translateY(-2px);
	}

	.teams-container {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
		gap: 1.5rem;
		margin-bottom: 2rem;
	}

	.team-card {
		background: white;
		border-radius: 20px;
		padding: 1.5rem;
		box-shadow: 0 8px 30px rgba(0, 0, 0, 0.2);
		transition: transform 0.3s ease, box-shadow 0.3s ease;
	}

	.team-card.trump-team {
		box-shadow:
			0 0 0 2px rgba(255, 215, 0, 0.7),
			0 0 18px rgba(255, 215, 0, 0.55),
			0 0 42px rgba(255, 215, 0, 0.35);
	}

	.team-card:hover {
		transform: translateY(-5px);
		box-shadow: 0 12px 40px rgba(0, 0, 0, 0.3);
	}

	.team1 {
		border-top: 5px solid #f093fb;
	}

	.team2 {
		border-top: 5px solid #4facfe;
	}

	.team-header {
		margin-bottom: 1rem;
	}

	.team-header h2 {
		margin: 0 0 1rem 0;
		color: #333;
		font-size: 1.5rem;
	}

	.level-display {
		display: flex;
		align-items: center;
		gap: 0.5rem;
		padding: 1rem;
		background: linear-gradient(135deg, #ffecd2 0%, #fcb69f 100%);
		border-radius: 15px;
		justify-content: center;
	}

	.level-label {
		font-size: 0.9rem;
		color: #666;
		font-weight: 600;
	}

	.level-card {
		font-size: 2.5rem;
		font-weight: bold;
		color: #d63031;
		text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.1);
		min-width: 50px;
		text-align: center;
	}

	.level-number {
		font-size: 0.9rem;
		color: #666;
	}

	.players {
		display: flex;
		flex-direction: column;
		gap: 0.75rem;
	}

	.player {
		padding: 0.75rem;
		background: #f8f9fa;
		border-radius: 10px;
		transition: all 0.2s ease;
	}

	.player span {
		color: #333;
		font-weight: 500;
		display: block;
	}

	.actions {
		display: flex;
		gap: 1rem;
		justify-content: center;
		align-items: center;
		margin-bottom: 2rem;
	}

	.score-btn,
	.reset-btn {
		padding: 1rem 2.5rem;
		font-size: 1.2rem;
		font-weight: bold;
		border: none;
		border-radius: 50px;
		cursor: pointer;
		transition: all 0.3s ease;
		box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
	}

	.score-btn {
		background: linear-gradient(135deg, #f093fb 0%, #f5576c 100%);
		color: white;
	}

	.score-btn:hover {
		transform: translateY(-3px);
		box-shadow: 0 6px 20px rgba(0, 0, 0, 0.3);
	}

	.reset-btn {
		background: linear-gradient(135deg, #4facfe 0%, #00f2fe 100%);
		color: white;
	}

	.reset-btn:hover {
		transform: translateY(-3px);
		box-shadow: 0 6px 20px rgba(0, 0, 0, 0.3);
	}

	.placements-grid {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(240px, 1fr));
		gap: 1.5rem;
		align-items: start;
	}

	.placements-team h4 {
		margin: 0 0 1rem 0;
		color: #667eea;
		font-size: 1.3rem;
		text-align: center;
	}

	.placement-row {
		display: flex;
		align-items: center;
		justify-content: space-between;
		gap: 1rem;
		padding: 0.75rem 1rem;
		background: rgba(102, 126, 234, 0.08);
		border-radius: 12px;
		margin-bottom: 0.75rem;
	}

	.placement-player {
		font-weight: 600;
		color: #333;
		flex: 1;
	}

	.placement-select {
		width: 90px;
		padding: 0.5rem 0.6rem;
		border-radius: 10px;
		border: 2px solid #e9ecef;
		background: white;
		font-weight: 700;
		text-align: center;
		outline: none;
	}

	.placements-actions {
		display: flex;
		justify-content: center;
		align-items: center;
		gap: 0.75rem;
		flex-wrap: wrap;
		margin-top: 1rem;
	}

	.share-btn {
		padding: 0.75rem 1.25rem;
		font-size: 1rem;
		font-weight: bold;
		border: none;
		border-radius: 50px;
		cursor: pointer;
		transition: all 0.3s ease;
		box-shadow: 0 4px 15px rgba(0, 0, 0, 0.15);
		background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
		color: white;
	}

	.share-btn:hover:enabled {
		transform: translateY(-2px);
		box-shadow: 0 6px 20px rgba(0, 0, 0, 0.2);
	}

	.share-btn:disabled {
		opacity: 0.7;
		cursor: not-allowed;
	}

	.export-error {
		color: #d63031;
		font-weight: 600;
		font-size: 0.95rem;
	}

	.placements-graph {
		margin-top: 1rem;
		margin-bottom: 1.5rem;
		background: white;
		border-radius: 15px;
		box-shadow: 0 4px 15px rgba(0, 0, 0, 0.08);
		padding: 0.75rem;
	}

	.placements-legend {
		display: flex;
		gap: 0.75rem 1.25rem;
		flex-wrap: wrap;
		justify-content: center;
		align-items: center;
		margin-top: 0.75rem;
		padding-top: 0.75rem;
		border-top: 1px solid rgba(0, 0, 0, 0.08);
	}

	.legend-item {
		display: flex;
		align-items: center;
		gap: 0.5rem;
	}

	.legend-line {
		width: 40px;
		height: 10px;
		display: block;
	}

	.legend-name {
		font-weight: 700;
		color: #333;
		font-size: 0.95rem;
	}

	.cancel-btn {
		margin-top: 1.5rem;
		padding: 0.8rem 2rem;
		font-size: 1rem;
		font-weight: bold;
		border: 2px solid #667eea;
		background: white;
		color: #667eea;
		border-radius: 25px;
		cursor: pointer;
		transition: all 0.3s ease;
		box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
	}

	.cancel-btn:hover {
		background: #667eea;
		color: white;
		transform: translateY(-2px);
		box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
	}

	.history-panel {
		background: white;
		border-radius: 20px;
		padding: 2rem;
		margin-top: 2rem;
		box-shadow: 0 8px 30px rgba(0, 0, 0, 0.2);
	}

	.history-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 1rem;
		flex-wrap: wrap;
		gap: 1rem;
	}

	.history-header h3 {
		margin: 0;
		color: #333;
		font-size: 1.5rem;
	}

	.undo-btn {
		padding: 0.5rem 1rem;
		font-size: 0.9rem;
		font-weight: bold;
		border: none;
		background: linear-gradient(135deg, #ffecd2 0%, #fcb69f 100%);
		color: #333;
		border-radius: 20px;
		cursor: pointer;
		transition: all 0.3s ease;
		box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
	}

	.undo-btn:hover {
		transform: translateY(-2px);
		box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
	}

	.history-table {
		overflow-x: auto;
	}

	.history-table table {
		width: 100%;
		border-collapse: separate;
		border-spacing: 0;
		box-shadow: 0 4px 15px rgba(0, 0, 0, 0.05);
		border-radius: 15px;
		overflow: hidden;
	}

	.history-table thead {
		background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
		color: white;
	}

	.history-table th {
		padding: 1rem;
		text-align: center;
		font-weight: bold;
		font-size: 1rem;
	}

	.history-table td {
		padding: 1rem;
		text-align: center;
		border-bottom: 1px solid #e9ecef;
		font-size: 0.95rem;
	}

	.history-table tbody tr {
		background: white;
		transition: background 0.2s ease;
	}

	.history-table tbody tr:last-child td {
		border-bottom: none;
	}

	.history-table tbody tr:hover {
		background: rgba(102, 126, 234, 0.05);
	}

	.info-modal-overlay {
		position: fixed;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		background: rgba(0, 0, 0, 0.6);
		display: flex;
		align-items: center;
		justify-content: center;
		z-index: 1000;
		padding: 1rem;
		animation: fadeIn 0.3s ease;
	}

	.info-modal {
		background: white;
		border-radius: 20px;
		padding: 0;
		max-width: 700px;
		width: 100%;
		max-height: 85vh;
		overflow: hidden;
		box-shadow: 0 10px 50px rgba(0, 0, 0, 0.3);
		animation: slideUp 0.3s ease;
		display: flex;
		flex-direction: column;
	}

	@keyframes slideUp {
		from {
			opacity: 0;
			transform: translateY(50px);
		}
		to {
			opacity: 1;
			transform: translateY(0);
		}
	}

	.info-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		padding: 1.5rem 2rem;
		border-bottom: 2px solid #667eea;
		background: linear-gradient(135deg, rgba(102, 126, 234, 0.1) 0%, rgba(118, 75, 162, 0.1) 100%);
	}

	.info-header h3 {
		margin: 0;
		color: #333;
		font-size: 1.8rem;
	}

	.close-btn {
		width: 40px;
		height: 40px;
		border: none;
		background: rgba(0, 0, 0, 0.1);
		border-radius: 50%;
		font-size: 1.5rem;
		cursor: pointer;
		transition: all 0.3s ease;
		display: flex;
		align-items: center;
		justify-content: center;
		color: #333;
	}

	.close-btn:hover {
		background: rgba(0, 0, 0, 0.2);
		transform: rotate(90deg);
	}

	.info-content {
		padding: 2rem;
		overflow-y: auto;
	}

	.scoring-modal {
		max-width: 900px;
	}

	.scoring-content {
		padding: 2rem;
		overflow-y: auto;
	}

	.scoring-actions {
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 0.75rem;
		margin-top: 1.5rem;
	}

	.scoring-actions .cancel-btn {
		margin-top: 0;
	}

	.options-modal {
		max-width: 500px;
	}

	.options-content {
		padding: 2rem;
	}

	.option-group {
		margin-bottom: 2rem;
	}

	.option-group:last-child {
		margin-bottom: 0;
	}

	.confirm-text {
		margin: 0;
		font-size: 1.1rem;
		line-height: 1.4;
		text-align: center;
		color: #333;
	}

	.confirm-actions {
		display: flex;
		justify-content: center;
		gap: 1rem;
		margin-top: 1.5rem;
		flex-wrap: wrap;
	}

	.option-label {
		display: flex;
		flex-direction: column;
		gap: 1rem;
	}

	.option-text {
		font-size: 1.2rem;
		font-weight: bold;
		color: #333;
	}

	.toggle-group {
		display: flex;
		gap: 0.5rem;
		background: #f0f0f0;
		padding: 0.3rem;
		border-radius: 25px;
	}

	.toggle-btn {
		flex: 1;
		padding: 0.75rem 1.5rem;
		font-size: 1rem;
		font-weight: 600;
		border: none;
		border-radius: 20px;
		cursor: pointer;
		transition: all 0.3s ease;
		background: transparent;
		color: #666;
	}

	.toggle-btn.active {
		background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
		color: white;
		box-shadow: 0 2px 10px rgba(102, 126, 234, 0.3);
	}

	.toggle-btn:hover:not(.active) {
		background: rgba(102, 126, 234, 0.1);
	}

	.info-content h4 {
		margin: 0 0 1.5rem 0;
		color: #667eea;
		font-size: 1.5rem;
		text-align: center;
	}


	.close-btn:hover {
		background: rgba(0, 0, 0, 0.2);
		transform: rotate(90deg);
	}

	/* Dark Mode Styles */
	.container.dark-mode {
		color: #e0e0e0;
	}

	.container.dark-mode .player {
		background: #2a2a3e;
	}

	.container.dark-mode .level-display {
		background: linear-gradient(135deg, rgba(74, 90, 154, 0.35) 0%, rgba(90, 74, 138, 0.35) 100%);
	}

	.container.dark-mode .placement-row {
		background: rgba(255, 255, 255, 0.06);
	}

	.container.dark-mode .placement-player {
		color: #e0e0e0;
	}

	.container.dark-mode .placement-select {
		background: #2a2a3e;
		border-color: #3a3a4e;
		color: #e0e0e0;
	}

	.container.dark-mode .placements-graph {
		background: #1e1e2e;
	}

	.container.dark-mode .placements-legend {
		border-top-color: #3a3a4e;
	}

	.container.dark-mode .legend-name {
		color: #e0e0e0;
	}

	.container.dark-mode .summary-table {
		box-shadow: 0 4px 15px rgba(0, 0, 0, 0.25);
	}

	.container.dark-mode .cancel-btn {
		background: #2a2a3e;
		border-color: #8b9aff;
		color: #e0e0e0;
	}

	.container.dark-mode .cancel-btn:hover {
		background: #8b9aff;
		color: #1e1e2e;
	}

	.container.dark-mode .confirm-text {
		color: #e0e0e0;
	}

	.dark-mode .mode-card,
	.dark-mode .team-card,
	.dark-mode .setup-team,
	.dark-mode .winner-card,
	.dark-mode .history-panel,
	.dark-mode .info-modal,
	.dark-mode .scoring-modal,
	.dark-mode .options-modal {
		background: #1e1e2e;
		color: #e0e0e0;
	}

	.dark-mode .mode-card h2 {
		color: #8b9aff;
	}

	.dark-mode .mode-card p,
	.dark-mode .subtitle {
		color: #b0b0b0;
	}

	.dark-mode .team-name-input,
	.dark-mode .setup-players input {
		background: #2a2a3e;
		border-color: #3a3a4e;
		color: #e0e0e0;
	}

	.dark-mode .team-name-input::placeholder,
	.dark-mode .setup-players input::placeholder {
		color: #808080;
	}

	.dark-mode .team-name-input:focus,
	.dark-mode .setup-players input:focus {
		border-color: #8b9aff;
		background: #333348;
	}

	.dark-mode h1,
	.dark-mode h2,
	.dark-mode h3,
	.dark-mode h4 {
		color: #e0e0e0;
	}

	.dark-mode .team-header h2,
	.dark-mode .winner-card h1,
	.dark-mode .winner-card h2 {
		color: #e0e0e0;
	}

	.dark-mode .level-label,
	.dark-mode .level-number {
		color: #b0b0b0;
	}

	.dark-mode .player span {
		color: #e0e0e0;
	}

	.dark-mode .history-table thead,
	.dark-mode .summary-table thead {
		background: linear-gradient(135deg, #4a5a9a 0%, #5a4a8a 100%);
	}

	.dark-mode .history-table tbody tr,
	.dark-mode .summary-table tbody tr {
		background: #1e1e2e;
	}

	.dark-mode .history-table tbody tr:hover,
	.dark-mode .summary-table tbody tr:hover {
		background: #2a2a3e;
	}

	.dark-mode .history-table td {
		color: #e0e0e0;
		border-bottom-color: #3a3a4e;
	}

	.dark-mode .info-header {
		background: linear-gradient(135deg, rgba(74, 90, 154, 0.3) 0%, rgba(90, 74, 138, 0.3) 100%);
	}

	.dark-mode .info-header h3,
	.dark-mode .history-header h3,
	.dark-mode .game-summary h3 {
		color: #e0e0e0;
	}

	.dark-mode .close-btn {
		background: rgba(255, 255, 255, 0.1);
		color: #e0e0e0;
	}

	.dark-mode .close-btn:hover {
		background: rgba(255, 255, 255, 0.2);
	}

	.dark-mode .option-text {
		color: #e0e0e0;
	}

	.dark-mode .toggle-group {
		background: #2a2a3e;
	}

	.dark-mode .toggle-btn {
		color: #b0b0b0;
	}

	.dark-mode .winner-stats {
		background: linear-gradient(135deg, rgba(74, 90, 154, 0.2) 0%, rgba(90, 74, 138, 0.2) 100%);
	}

	.dark-mode .stat-label {
		color: #b0b0b0;
	}

	.dark-mode .stat-value {
		color: #8b9aff;
	}

	@media (max-width: 768px) {
		.container {
			padding: 1rem;
		}

		h1 {
			font-size: 2rem;
		}

		.mode-select-screen {
			min-height: 100vh;
			padding: 1rem 0;
		}

		.mode-select-screen h1 {
			font-size: 2.5rem;
		}

		.mode-header {
			margin-bottom: 2rem;
			max-width: 100%;
			padding: 0 1rem;
		}

		.mode-card {
			padding: 2rem 1.5rem;
		}

		.mode-icon {
			font-size: 3rem;
		}

		.mode-card h2 {
			font-size: 2rem;
		}

		.mode-card p {
			font-size: 1rem;
		}

		.teams-container {
			grid-template-columns: 1fr;
		}

		.actions {
			flex-direction: column;
			align-items: center;
		}

		.placements-grid {
			grid-template-columns: 1fr;
		}
	}
</style>
