<script lang="ts">
	import { onMount } from 'svelte';

	type GameMode = '2v2' | '3v3';
	type Screen = 'mode-select' | 'team-setup' | 'game' | 'winner';
	type WinningTeam = 1 | 2 | null;
	type RoundHistory = {
		round: number;
		winner: string;
		combo: string;
		points: number;
		team1Level: number;
		team2Level: number;
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
	let selectedWinner = $state<WinningTeam>(null);
	let roundHistory = $state<RoundHistory[]>([]);
	let roundNumber = $state(1);
	let showScoringInfo = $state(false);
	let isInitialSetup = $state(true);
	let gameWinner = $state<string>('');
	let gameDuration = $state(0);

	const levelCards = ['2', '3', '4', '5', '6', '7', '8', '9', '10', 'J', 'Q', 'K', 'A'];

	function getLevelCard(level: number): string {
		if (level < 2) return '2';
		if (level > 14) return 'A';
		return levelCards[level - 2];
	}

	// Scoring combos for 2v2 mode
	const combos2v2 = [
		{ name: '双下', positions: '1-2', points: 3, description: 'Both finish 1st and 2nd' },
		{ name: '单下', positions: '1-3', points: 2, description: 'One finishes 1st, other 3rd' },
		{ name: '单下', positions: '1-4', points: 2, description: 'One finishes 1st, other 4th' },
		{ name: '单上', positions: '2-3', points: 1, description: 'Finish 2nd and 3rd' },
		{ name: '单上', positions: '2-4', points: 1, description: 'Finish 2nd and 4th' }
	];

	// Scoring combos for 3v3 mode
	const combos3v3 = [
		{ name: '三下', positions: '1-2-3', points: 4, description: 'All three finish 1st, 2nd, 3rd' },
		{ name: '两下半', positions: '1-2-4', points: 3, description: 'Two finish 1st and 2nd' },
		{ name: '两下半', positions: '1-2-5', points: 3, description: 'Two finish 1st and 2nd' },
		{ name: '两下半', positions: '1-2-6', points: 3, description: 'Two finish 1st and 2nd' },
		{ name: '单下', positions: '1-3-4', points: 2, description: 'One finishes 1st' },
		{ name: '单下', positions: '1-3-5', points: 2, description: 'One finishes 1st' },
		{ name: '单下', positions: '1-3-6', points: 2, description: 'One finishes 1st' },
		{ name: '单下', positions: '1-4-5', points: 2, description: 'One finishes 1st' },
		{ name: '单下', positions: '1-4-6', points: 2, description: 'One finishes 1st' },
		{ name: '单下', positions: '1-5-6', points: 2, description: 'One finishes 1st' },
		{ name: '单上', positions: '2-3-4', points: 1, description: 'None finish 1st' },
		{ name: '单上', positions: '2-3-5', points: 1, description: 'None finish 1st' },
		{ name: '单上', positions: '2-3-6', points: 1, description: 'None finish 1st' },
		{ name: '单上', positions: '2-4-5', points: 1, description: 'None finish 1st' },
		{ name: '单上', positions: '2-4-6', points: 1, description: 'None finish 1st' },
		{ name: '单上', positions: '2-5-6', points: 1, description: 'None finish 1st' }
	];

	function getCurrentCombos() {
		return gameMode === '2v2' ? combos2v2 : combos3v3;
	}

	function selectMode(mode: GameMode) {
		gameMode = mode;
		// Reset player arrays based on mode
		if (mode === '2v2') {
			team1Players = ['Player 1', 'Player 2'];
			team2Players = ['Player 3', 'Player 4'];
		} else {
			team1Players = ['Player 1', 'Player 2', 'Player 3'];
			team2Players = ['Player 4', 'Player 5', 'Player 6'];
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

	function selectWinningTeam(team: 1 | 2) {
		selectedWinner = team;
	}

	function checkWinCondition(combo: { name: string; positions: string; points: number }) {
		// Win condition: be on level A (14) AND win with no dweller
		// No dweller means:
		// - 2v2: positions must be "1-2" (双下)
		// - 3v3: positions must be "1-2-3" (三下)
		
		const isOnAce = (selectedWinner === 1 && team1Level === 14) || (selectedWinner === 2 && team2Level === 14);
		
		if (!isOnAce) return false;
		
		const noDweller = (gameMode === '2v2' && combo.positions === '1-2') || 
		                   (gameMode === '3v3' && combo.positions === '1-2-3');
		
		if (noDweller) {
			gameWinner = selectedWinner === 1 ? team1Name : team2Name;
			gameDuration = roundNumber - 1;
			currentScreen = 'winner';
			return true;
		}
		
		return false;
	}

	function scoreGame(combo: { name: string; positions: string; points: number }) {
		if (selectedWinner === null) return;

		const winnerName = selectedWinner === 1 ? team1Name : team2Name;
		const points = combo.points;

		// Check for win condition BEFORE updating levels
		// (team must be on A and win with no dweller)
		const wonGame = checkWinCondition(combo);

		// Add to history
		roundHistory.push({
			round: roundNumber,
			winner: winnerName,
			combo: `${combo.name} (${combo.positions})`,
			points: points,
			team1Level: team1Level,
			team2Level: team2Level
		});

		// Update levels
		if (selectedWinner === 1) {
			team1Level += points;
		} else {
			team2Level += points;
		}

		// If won, show winner screen
		if (wonGame) {
			return;
		}

		// Reset and close
		roundNumber++;
		selectedWinner = null;
		showScoring = false;
	}

	function cancelScoring() {
		selectedWinner = null;
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
		selectedWinner = null;
		currentScreen = 'mode-select';
		showScoring = false;
		isInitialSetup = true;
		gameWinner = '';
		gameDuration = 0;
	}

	function undoLastRound() {
		if (roundHistory.length === 0) return;
		
		const lastRound = roundHistory[roundHistory.length - 1];
		team1Level = lastRound.team1Level;
		team2Level = lastRound.team2Level;
		roundHistory.pop();
		roundNumber--;
	}
</script>

<div class="container">
	{#if currentScreen === 'mode-select'}
		<!-- Mode Selection Screen -->
		<div class="screen-content mode-select-screen">
			<h1>🎴 Guandan Scorer</h1>
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
				{/if}

				<button class="new-game-btn" onclick={resetGame}>🎴 New Game</button>
			</div>
		</div>
	{:else}
		<!-- Game Screen -->
		<div class="screen-content game-screen">
			<header>
				<h1>🎴 Guandan Scorer</h1>
				<button class="edit-btn" onclick={backToSetup}>✏️ Edit Teams</button>
			</header>

			<div class="teams-container">
				<!-- Team 1 -->
				<div class="team-card team1">
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
				<div class="team-card team2">
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
				<button class="score-btn" onclick={() => (showScoring = !showScoring)}>
					📊 Score Round
				</button>
				<button class="info-btn" onclick={() => (showScoringInfo = !showScoringInfo)}>
					ℹ️ Scoring Info
				</button>
				<button class="reset-btn" onclick={resetGame}>🔄 Reset Game</button>
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

			<!-- Scoring Info Modal -->
			{#if showScoringInfo}
				<div class="info-modal-overlay" onclick={() => (showScoringInfo = false)} onkeydown={(e) => e.key === 'Escape' && (showScoringInfo = false)} role="button" tabindex="0">
					<div class="info-modal" onclick={(e) => e.stopPropagation()} onkeydown={(e) => e.stopPropagation()} role="dialog" tabindex="-1">
						<div class="info-header">
							<h3>Scoring Rules</h3>
							<button class="close-btn" onclick={() => (showScoringInfo = false)}>✕</button>
						</div>
						<div class="info-content">
							{#if gameMode === '2v2'}
								<h4>2v2 Mode</h4>
								<table class="info-table">
									<thead>
										<tr>
											<th>Combo</th>
											<th>Positions</th>
											<th>Points</th>
										</tr>
									</thead>
									<tbody>
										{#each combos2v2 as combo}
											<tr>
												<td>{combo.name}</td>
												<td>{combo.positions}</td>
												<td>+{combo.points}</td>
											</tr>
										{/each}
									</tbody>
								</table>
							{:else}
								<h4>3v3 Mode</h4>
								<table class="info-table">
									<thead>
										<tr>
											<th>Combo</th>
											<th>Positions</th>
											<th>Points</th>
										</tr>
									</thead>
									<tbody>
										{#each combos3v3 as combo}
											<tr>
												<td>{combo.name}</td>
												<td>{combo.positions}</td>
												<td>+{combo.points}</td>
											</tr>
										{/each}
									</tbody>
								</table>
							{/if}
						</div>
					</div>
				</div>
			{/if}

			<!-- Scoring Modal -->
			{#if showScoring}
				<div class="info-modal-overlay" onclick={() => (showScoring = false, selectedWinner = null)} onkeydown={(e) => e.key === 'Escape' && (showScoring = false, selectedWinner = null)} role="button" tabindex="0">
					<div class="info-modal scoring-modal" onclick={(e) => e.stopPropagation()} onkeydown={(e) => e.stopPropagation()} role="dialog" tabindex="-1">
						<div class="info-header">
							<h3>{selectedWinner === null ? 'Select Winning Team' : `Select Combo for ${selectedWinner === 1 ? team1Name : team2Name}`}</h3>
							<button class="close-btn" onclick={() => (showScoring = false, selectedWinner = null)}>✕</button>
						</div>
						<div class="info-content scoring-content">
							{#if selectedWinner === null}
								<div class="team-selection">
									<button class="team-select-btn team1" onclick={() => selectWinningTeam(1)}>
										<span class="team-name">{team1Name}</span>
										<span class="team-level">Level {getLevelCard(team1Level)}</span>
									</button>
									<button class="team-select-btn team2" onclick={() => selectWinningTeam(2)}>
										<span class="team-name">{team2Name}</span>
										<span class="team-level">Level {getLevelCard(team2Level)}</span>
									</button>
								</div>
							{:else}
								<div class="combo-grid">
									{#each getCurrentCombos() as combo}
										<button class="combo-btn" onclick={() => scoreGame(combo)}>
											<span class="combo-name">{combo.name}</span>
											<span class="combo-positions">{combo.positions}</span>
											<span class="combo-points">+{combo.points}</span>
										</button>
									{/each}
								</div>
								<button class="cancel-btn" onclick={() => (selectedWinner = null)}>← Back</button>
							{/if}
						</div>
					</div>
				</div>
			{/if}
		</div>
	{/if}
</div>

<style>
	:global(body) {
		margin: 0;
		padding: 0;
		font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu,
			Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
		background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
		min-height: 100vh;
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

	.mode-select-screen h1 {
		color: white;
		font-size: 4rem;
		margin: 0 0 3rem 0;
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
		justify-content: space-between;
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

	.team-selection {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
		gap: 1.5rem;
		margin-top: 1rem;
	}

	.team-select-btn {
		padding: 2rem;
		border: none;
		border-radius: 20px;
		cursor: pointer;
		transition: all 0.3s ease;
		box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
		display: flex;
		flex-direction: column;
		gap: 0.5rem;
		align-items: center;
		justify-content: center;
	}

	.team-select-btn.team1 {
		background: linear-gradient(135deg, #f093fb 0%, #f5576c 100%);
		color: white;
	}

	.team-select-btn.team2 {
		background: linear-gradient(135deg, #4facfe 0%, #00f2fe 100%);
		color: white;
	}

	.team-select-btn:hover {
		transform: translateY(-5px) scale(1.02);
		box-shadow: 0 8px 25px rgba(0, 0, 0, 0.2);
	}

	.team-name {
		font-size: 1.5rem;
		font-weight: bold;
	}

	.team-level {
		font-size: 1.1rem;
		opacity: 0.9;
	}

	.combo-grid {
		display: grid;
		grid-template-columns: repeat(auto-fill, minmax(140px, 1fr));
		gap: 1rem;
		margin-top: 1.5rem;
	}

	.combo-btn {
		padding: 1.5rem 1rem;
		background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
		border: none;
		border-radius: 15px;
		cursor: pointer;
		transition: all 0.3s ease;
		box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
		color: white;
		display: flex;
		flex-direction: column;
		gap: 0.5rem;
		align-items: center;
		justify-content: center;
	}

	.combo-btn:hover {
		transform: translateY(-5px);
		box-shadow: 0 8px 25px rgba(0, 0, 0, 0.2);
	}

	.combo-name {
		font-size: 1.3rem;
		font-weight: bold;
		display: block;
	}

	.combo-positions {
		font-size: 1rem;
		opacity: 0.9;
		display: block;
	}

	.combo-points {
		font-size: 1.1rem;
		background: rgba(255, 255, 255, 0.3);
		padding: 0.4rem 0.8rem;
		border-radius: 15px;
		display: block;
		font-weight: bold;
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

	.info-btn {
		padding: 1rem 2.5rem;
		font-size: 1.2rem;
		font-weight: bold;
		border: none;
		border-radius: 50px;
		cursor: pointer;
		transition: all 0.3s ease;
		box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
		background: linear-gradient(135deg, #ffecd2 0%, #fcb69f 100%);
		color: #333;
	}

	.info-btn:hover {
		transform: translateY(-3px);
		box-shadow: 0 6px 20px rgba(0, 0, 0, 0.3);
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

	.info-content h4 {
		margin: 0 0 1.5rem 0;
		color: #667eea;
		font-size: 1.5rem;
		text-align: center;
	}

	.info-table {
		width: 100%;
		border-collapse: separate;
		border-spacing: 0;
		box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
		border-radius: 15px;
		overflow: hidden;
	}

	.info-table thead {
		background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
		color: white;
	}

	.info-table th,
	.info-table td {
		padding: 1rem;
		text-align: center;
	}

	.info-table th {
		font-weight: bold;
		font-size: 1.1rem;
	}

	.info-table tbody tr {
		background: white;
		border-bottom: 1px solid #e9ecef;
	}

	.info-table tbody tr:last-child {
		border-bottom: none;
	}

	.info-table tbody tr:hover {
		background: rgba(102, 126, 234, 0.05);
	}

	.info-table td:first-child {
		font-weight: bold;
		color: #667eea;
		font-size: 1.1rem;
	}

	.close-btn:hover {
		background: rgba(0, 0, 0, 0.2);
		transform: rotate(90deg);
	}

	@media (max-width: 768px) {
		.container {
			padding: 1rem;
		}

		h1 {
			font-size: 2rem;
		}

		.teams-container {
			grid-template-columns: 1fr;
		}

		.actions {
			flex-direction: column;
		}

		.combo-grid {
			grid-template-columns: repeat(auto-fill, minmax(120px, 1fr));
		}
	}
</style>
