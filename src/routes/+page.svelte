<script>
	import { onMount } from 'svelte';
	let data = [];

	let allPosts = [];
	let currentPage = 1;
	let postsPerPage = 1;
	let totalPosts = 0;
	let totalPages = 0;

	onMount(async () => {
		const sheetId = '1ktrTyIcfjtdgatYKf5E5hi3gOBzUr2bHnXgfmLFpG8c';
		const sheetName = encodeURIComponent('Leaderboard');
		const sheetURL = `https://docs.google.com/spreadsheets/d/${sheetId}/gviz/tq?tqx=out:csv&sheet=${sheetName}`;
		const response = await fetch(sheetURL);

		const csv = await response.text();

		const [header, ...rows] = csv.split('\n');
		const keys = header.split(',').map((key) => key.replace(/"/g, ''));

		// Parse the CSV data
		data = rows.map((row) => {
			const values = row.split(',').map((value) => value.replace(/"/g, ''));
			const keysWithoutQuotes = keys.map((key) => key.replace(/"/g, ''));
			let obj = keysWithoutQuotes.reduce((obj, key, i) => ({ ...obj, [key]: values[i] }), {});
			if (obj['PhotoURL'] && obj['PhotoURL'].startsWith('https://drive.google.com')) {
				let url = new URL(obj['PhotoURL']);
				let pathParts = url.pathname.split('/');
				obj['Photo'] = pathParts[3]; // The ID is the 4th part of the path
			}
			if (obj['TeamURL'] && obj['TeamURL'].startsWith('https://drive.google.com')) {
				let url = new URL(obj['TeamURL']);
				let pathParts = url.pathname.split('/');
				obj['Team'] = pathParts[3]; // The ID is the 4th part of the path
			}
			return obj;
		});

		let periods = new Map();
		let teamCash = new Map();
		let teamData = new Map();
		let teamMembersCount = new Map();
		let teamAverageCash = new Map();
		let teamTotalCash = new Map();

		for (let item of data) {
			// Add the period to the team
			let periodKey = `${item.Period}`;
			if (!periods.has(periodKey)) {
				periods.set(periodKey, new Map());
			}

			// Add the team to the period
			let teams = periods.get(periodKey);
			let teamKey = `${item.TeamName}`;
			if (!teams.has(teamKey)) {
				teams.set(teamKey, item);
			}

			// Calculate the total cash for each team
			let cashKey = `${item.TeamName}-${item.Period}`;
			if (teamCash.has(cashKey)) {
				teamCash.set(cashKey, teamCash.get(cashKey) + Number(item.Cash));
			} else {
				teamCash.set(cashKey, Number(item.Cash));
			}

			// Count the number of members in each team
			if (teamMembersCount.has(cashKey)) {
				teamMembersCount.set(cashKey, teamMembersCount.get(cashKey) + 1);
			} else {
				teamMembersCount.set(cashKey, 1);
			}

			// Combine all student data by period and team
			let dataKey = `${item.TeamName}-${item.Period}`;
			if (teamData.has(dataKey)) {
				teamData.get(dataKey).push(item);
			} else {
				teamData.set(dataKey, [item]);
			}
		}

		// Calculate the total cash for each team
		teamCash.forEach((cash, key) => {
			let totalCash =+ cash;
			teamTotalCash.set(key, totalCash);
		});

		// Calculate the average cash per person for each team
		teamCash.forEach((totalCash, key) => {
			let averageCash = totalCash / teamMembersCount.get(key);
			teamAverageCash.set(key, averageCash);
		});

		let result = Array.from(periods.values()).map((teams) => {
			let teamArray = Array.from(teams.values());
			teamArray.sort(
				(a, b) =>
					teamAverageCash.get(`${b.TeamName}-${b.Period}`) - teamAverageCash.get(`${a.TeamName}-${a.Period}`)
			);
			return teamArray;
		});

		// Add the combined student data to each team entry
		for (let period of result) {
			for (let team of period) {
				team.students = teamData.get(`${team.TeamName}-${team.Period}`);
				team.totalTeamCash = teamTotalCash.get(`${team.TeamName}-${team.Period}`);

				console.log(team.totalTeamCash, `${team.TeamName}-${team.Period}`);
			}
		}

		allPosts = result;
		totalPosts = allPosts.length;
		totalPages = Math.ceil(totalPosts / postsPerPage);
	});

	$: postRangeHigh = currentPage * postsPerPage;
	$: postRangeLow = postRangeHigh - postsPerPage;

	const setCurrentPage = (newPage) => {
		currentPage = newPage;
	};

	function periodAdjustment (currentPage) {
		switch(currentPage) {
  			case 1:
				return 1;
  			case 2:
				return 3;
  			case 3:
				return 4;
  			case 4:
				return 5;
  			case 5:
				return 7;
		}
	}
</script>

<div id="pagination">
<h1>Leaderboard - Period </h1>
<ul>
	{#each [4, 3, 2, 1] as i}
		{#if currentPage - i > 0}
			<li>
				<a
					href="/period/{currentPage - i}"
					on:click|preventDefault={() => setCurrentPage(currentPage - i)}>{periodAdjustment(currentPage - i)}</a
				>
			</li>
		{/if}
	{/each}
	<li><span>{periodAdjustment(currentPage)}</span></li>
	{#each Array(4) as _, i}
		{#if currentPage + (i + 1) <= totalPages}
			<li>
				<a
					href="/period/{currentPage + (i + 1)}"
					on:click|preventDefault={() => setCurrentPage(currentPage + (i + 1))}
					>{periodAdjustment(currentPage + (i + 1))}</a
				>
			</li>
		{/if}
	{/each}
</ul>
</div>

{#each allPosts as post, i}
	{#if i >= postRangeLow && i < postRangeHigh}
		<ul class="teams">
			{#each allPosts[currentPage - 1] as team (team.TeamName)}
				<li>
					<img src="https://drive.google.com/thumbnail?id={team.Team}" alt={team.Name} />
					<div class="team-data">
						<div class="team-cash">${team.totalTeamCash}</div>
                        <h3>{team.TeamName}</h3>
                        {#each team.students as student (student.Name)}
                            <div class="team">
                                <img src="https://drive.google.com/thumbnail?id={student.Photo}" alt={student.Name} />
                                <p>{student.Name} <span>${student.Cash}</span></p>
                            </div>
                        {/each}
                    </div>
				</li>
			{/each}
		</ul>
	{/if}
{/each}

<style>
	.teams {
		display: flex;
		flex-direction: row;
		flex-wrap: wrap;
		gap: 20px;
		list-style: none;
		justify-content: center;
		margin: 0;
		padding: 0;
	}
    .teams img {
        border-top-left-radius: 20px;
    }
    .teams > li {
		flex: none;
		width: calc(100% - 6);
        position: relative;
        display: flex;
        flex-direction: column;
        align-items: stretch;
        border-top-left-radius: 20px;
        border-bottom-right-radius: 20px;
        box-shadow: 2px 5px 20px 5px hsla(255, 50%, 61%, 0.35),
                    -2px -5px 20px 5px hsla(255, 50%, 61%, 0.35);
                    
        background-image: url('$lib/assets/images/placeholder-square.jpg');
    }
    .teams > li .team-data {
        padding: 10px;
        border-bottom-right-radius: 20px;
        flex: 1;
		position: relative;
    }
	.teams .team-cash {
		position: absolute;
		top: 0;
		left: -2px;
		background-color: hsl(117, 55%, 35%);
		transform: translateY(calc(-10px + -100%));
		padding: 8px 12px;
		color: white;
		filter: drop-shadow(5px 5px 5px hsla(0, 0%, 0%, 0.95));
	}
    .teams .team-data h3 {
        margin-top: 0;
        color: white;
        font-size: 1.25rem;
        font-weight: bold;
        text-align: center;
        background-color: hsla(0, 100%, 100%, 0.15);
        padding: 8px 0 5px;
        border-bottom: 5px double hsla(0, 100%, 100%, 0.5);
		margin-bottom: 1rem;
    }
    .teams .team-data p {
        color: white;
		display: flex;
		width: 100%;
		justify-content: space-between;
    }
	.teams .team-data p span {
		/* display: none; */
		margin-left: auto;
	}
    .teams li:nth-child(1) .team-data {
        background-color: hsl(14, 61%, 43%);
    }
    .teams li:nth-child(1)::before {
        content: '';
        position: absolute;
        top: 0;
        right: 0;
        width: 100%;
        height: 200px;
        background-image: url('$lib/assets/images/first.png');
        background-position: top right;
        background-repeat: no-repeat;
        background-size: 65%;
        filter: drop-shadow(5px 5px 5px hsla(0, 0%, 0%, 0.95));
    }
    .teams li:nth-child(2) .team-data {
        background-color: rgb(181, 158, 57);
    }
	.teams li:nth-child(3) .team-data {
        background-color: rgb(54, 131, 161);
    }
	.teams li:nth-child(4) .team-data {
        background-color: rgb(85, 161, 81);
    }
	.teams li:nth-child(5) .team-data {
        background-color: rgb(79, 64, 158);
    }
	.teams li:nth-child(6) .team-data {
        background-color: rgb(148, 80, 150);
    }
    .team {
        display: flex;
        align-items: center;
        margin-bottom: 5px;
    }
    .team img {
        width: 1.5rem;
        height: 1.5rem;
        margin-right: 0.5rem;
        border-radius: 1.5rem;
		padding: 3px;
		border-width: 2px;
		border-style: solid;
		border-color: hsla(0, 100%, 100%, 0.5);
		background-color: hsla(0, 100%, 100%, 0.15);
    }
    .team p {
        font-size: 1rem;
        margin: 0;
    }
    #pagination {
        display: flex;
        align-items: center;
        justify-content: center;
        margin-bottom: 2rem;
    }
    #pagination h1,
    #pagination a,
    #pagination span {
        margin: 0;
        font-size: 2rem;
        font-weight: bold;
        color: lightgray;
    }
	#pagination ul {
		display: flex;
        list-style: none;
        margin: 0;
        padding: 0;
        gap: 1rem;
        margin-left: 0.5rem;
	}
	#pagination a,
	#pagination span {
		text-decoration: none;
        color: #7941b1;
	}
	#pagination span {
		color: #fa55d4;
	}
	@font-face {
		font-family: 'Arvo';
		font-style: normal;
		font-weight: 500;
		src: url('$lib/assets/fonts/Arvo-Regular.ttf') format('truetype');
	}
	:global(html),
	:global(body) {
		min-height: 100%;
	}
	:global(body) {
		background-image: url('$lib/assets/images/retro-animated-background.gif');
		background-position: center;
		background-size: cover;
		background-repeat: no-repeat;
		font-family: 'Arvo', sans-serif;
	}
</style>
