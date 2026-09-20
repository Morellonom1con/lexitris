<script lang="ts">
	import { onMount } from "svelte";
	type Cell = { className: string; letter: string };
	const maxGuesses = 4;

	let validGuesses: Set<string>;
	let validTargets: string[];

	let board: number[][] = $state(
		Array.from({ length: 24 }, () => Array(10).fill(0)),
	);
	let phase: "building" | "falling" | "gameover" = $state("building");

	let target: string = $state("NOTCH");
	let guesses: string[] = $state([]);
	let newGuess = $state("");

	let heldWord = $state("");
	let holdCharge = $state(1);
	let rerollCharge = $state(1);

	let frameOrigin = $state([0, 2]);
	let currBlock: string[][] = $state([]);
	let rotationAnchor: number[] = $state([]);

	let greens = $derived.by(() => {
		let greens = new Set<string>();
		for (let i = 0; i < guesses.length; i++) {
			for (let j = 0; j < 5; j++) {
				if (guesses[i][j] == target[j]) {
					greens.add(`${i},${j}`);
				}
			}
		}
		return greens;
	});
	let currentLongest = $derived(largestConnected(greens));
	let currBlockOrigin: number[] = $state([0, 0]);
	let displayGrid: Cell[][] = $derived(
		updateGrid(
			board,
			phase,
			guesses,
			frameOrigin,
			currBlockOrigin,
			target,
			currBlock,
		),
	);

	onMount(async () => {
		const validNonAnswers = new Set(
			(await (await fetch("/guesses.txt")).text()).split(
				"\n",
			),
		);
		const validAnswers = new Set(
			(await (await fetch("/answers.txt")).text()).split(
				"\n",
			),
		);
		validGuesses = new Set(validNonAnswers.union(validAnswers));
		validTargets = [...validAnswers];
		target = getNewTarget();
	});
	onMount(() => {
		const interval = setInterval(() => {
			if (phase == "building") {
				if (
					canFrameFall(
						phase,
						frameOrigin,
						currBlockOrigin,
						guesses,
						board,
						currBlock,
					)
				) {
					frameOrigin[0]++;
				} else {
					currBlock = getBlock(currentLongest);
					currBlockOrigin =
						getBoundingRect(
							currentLongest,
						)[0];
					phase = "falling";
					if (currBlock.length == 0) {
						phase = "gameover";
					} else {
						rotationAnchor = [
							currBlockOrigin[0] +
								currBlock.length /
									2,
							currBlockOrigin[1] +
								currBlock[0]
									.length /
									2,
						];
					}
				}
			} else if (phase == "falling") {
				if (
					canFrameFall(
						phase,
						frameOrigin,
						currBlockOrigin,
						guesses,
						board,
						currBlock,
					)
				) {
					frameOrigin[0]++;
				} else {
					lockBlock(
						currBlock,
						frameOrigin,
						currBlockOrigin,
					);
				}
			}
		}, 1000);
		return () => clearInterval(interval);
	});
	function rotateBlock(currBlock: string[][]) {
		let result: string[][] = [];
		for (let j = 0; j < currBlock[0].length; j++) {
			const newRow = [];
			for (let i = currBlock.length - 1; i >= 0; i--) {
				newRow.push(currBlock[i][j]);
			}
			result.push(newRow);
		}
		return result;
	}

	function isValidBlockPosition(
		currBlock: string[][],
		frameOrigin: number[],
		currBlockOrigin: number[],
		board: number[][],
	) {
		if (
			frameOrigin[0] + currBlockOrigin[0] >= 0 &&
			frameOrigin[1] + currBlockOrigin[1] >= 0 &&
			frameOrigin[0] +
				currBlockOrigin[0] +
				currBlock.length -
				1 <
				board.length &&
			frameOrigin[1] +
				currBlockOrigin[1] +
				currBlock[0].length -
				1 <
				board[0].length
		) {
			for (let i = 0; i < currBlock.length; i++) {
				for (let j = 0; j < currBlock[0].length; j++) {
					let r =
						i +
						currBlockOrigin[0] +
						frameOrigin[0];
					let c =
						j +
						currBlockOrigin[1] +
						frameOrigin[1];
					if (
						currBlock[i][j] == "1" &&
						board[r][c] == 1
					) {
						return false;
					}
				}
			}
			return true;
		} else return false;
	}
	function lockBlock(
		currBlock: string[][],
		frameOrigin: number[],
		currBlockOrigin: number[],
	) {
		for (let i = 0; i < currBlock.length; i++) {
			for (let j = 0; j < currBlock[0].length; j++) {
				if (currBlock[i][j] == "1")
					board[
						i +
							frameOrigin[0] +
							currBlockOrigin[0]
					][
						j +
							frameOrigin[1] +
							currBlockOrigin[1]
					] = 1;
			}
		}
		for (let i = 0; i < board.length; i++) {
			if (board[i].every((val) => val === 1)) {
				for (let j = i; j > 0; j--) {
					board[j] = board[j - 1];
				}
				board[0] = Array(10).fill(0);
			}
		}
		for (let i = 0; i < 4; i++) {
			for (let j = 0; j < board[0].length; j++) {
				if (board[i][j] == 1) {
					phase = "gameover";
					return;
				}
			}
		}
		nextBuild();
	}
	function updateGrid(
		board: number[][],
		phase: "building" | "falling" | "gameover",
		guesses: string[],
		frameOrigin: number[],
		currBlockOrigin: number[],
		target: string,
		currBlock: string[][],
	) {
		let tempGrid: Cell[][] = Array.from({ length: 24 }, () =>
			Array.from({ length: 10 }, () => ({
				className: "",
				letter: "",
			})),
		);
		for (let i = 0; i < board.length; i++) {
			for (let j = 0; j < board[0].length; j++) {
				let className = "";
				let letter = "";
				if (phase == "building") {
					if (withinFrame(frameOrigin, [i, j])) {
						let r = i - frameOrigin[0];
						let c = j - frameOrigin[1];
						if (
							guesses[r] &&
							guesses[r][c] ===
								target[c]
						) {
							className = "greenBox";
							letter = guesses[r][c];
						} else if (guesses[r]) {
							className = "greyBox";
							letter = guesses[r][c];
						} else {
							className = "nullBox";
						}
						tempGrid[i][j].className =
							className;

						tempGrid[i][j].letter = letter;
						continue;
					}
				}
				if (phase == "falling") {
					let r =
						i -
						currBlockOrigin[0] -
						frameOrigin[0];
					let c =
						j -
						currBlockOrigin[1] -
						frameOrigin[1];
					if (
						currBlock[r] &&
						currBlock[r][c] === "1"
					) {
						className = "greenBox";
						tempGrid[i][j].className =
							className;
						continue;
					}
				}

				if (i < 4) {
					className += "grace";
				} else {
					className += "tetris";
				}
				if (board[i][j] == 1) {
					className += "FilledBox";
				} else {
					className += "UnfilledBox";
				}
				tempGrid[i][j].className = className;
			}
		}
		return tempGrid;
	}
	function getNewTarget() {
		const roll = Math.random();
		const targetIndex = roll * validTargets.length;
		return validTargets[Math.floor(targetIndex)].toUpperCase();
	}
	function canFrameFall(
		phase: "building" | "falling" | "gameover",
		frameOrigin: number[],
		currBlockOrigin: number[],
		guesses: string[],
		board: number[][],
		currBlock: string[][],
	) {
		if (phase == "building") {
			const row = guesses.length + frameOrigin[0];
			for (
				let i = frameOrigin[1];
				i < frameOrigin[1] + 5;
				i++
			) {
				if (row < 24) {
					if (board[row][i] == 1) return false;
				} else return false;
			}
			return true;
		} else if (phase == "falling") {
			let cellsToBeChecked: number[][] = [];
			for (let j = 0; j < currBlock[0].length; j++) {
				let i = currBlock.length;
				while (
					i - 1 > 0 &&
					currBlock[i - 1][j] == "0"
				) {
					i--;
				}
				cellsToBeChecked.push([i, j]);
			}
			for (let i = 0; i < cellsToBeChecked.length; i++) {
				if (
					cellsToBeChecked[i][0] +
						currBlockOrigin[0] +
						frameOrigin[0] <
					24
				) {
					if (
						board[
							cellsToBeChecked[i][0] +
								currBlockOrigin[0] +
								frameOrigin[0]
						][
							cellsToBeChecked[i][1] +
								currBlockOrigin[1] +
								frameOrigin[1]
						] == 1
					)
						return false;
				} else return false;
			}
			return true;
		}
	}
	function canFrameMoveRight(
		guessesBoundingRectInBoardSpace: number[][],
		board: number[][],
	) {
		let j = guessesBoundingRectInBoardSpace[1][1];
		for (
			let i = guessesBoundingRectInBoardSpace[0][0];
			i <= guessesBoundingRectInBoardSpace[1][0];
			i++
		) {
			if (board[i][j + 1] == 1) return false;
		}
		return true;
	}
	function canFrameMoveLeft(
		guessesBoundingRectInBoardSpace: number[][],
		board: number[][],
	) {
		let j = guessesBoundingRectInBoardSpace[0][1];
		for (
			let i = guessesBoundingRectInBoardSpace[0][0];
			i <= guessesBoundingRectInBoardSpace[1][0];
			i++
		) {
			if (board[i][j - 1] == 1) return false;
		}
		return true;
	}
	function canBlockMoveRight(
		currBlock: string[][],
		currBlockOrigin: number[],
		frameOrigin: number[],
		board: number[][],
	) {
		let cellsToBeChecked: number[][] = [];
		for (let i = 0; i < currBlock.length; i++) {
			let j = currBlock[0].length;
			while (j - 1 > 0 && currBlock[i][j - 1] == "0") {
				j--;
			}
			cellsToBeChecked.push([i, j]);
		}
		for (let i = 0; i < cellsToBeChecked.length; i++) {
			if (
				cellsToBeChecked[i][1] +
					currBlockOrigin[1] +
					frameOrigin[1] <
				board[0].length
			) {
				if (
					board[
						cellsToBeChecked[i][0] +
							currBlockOrigin[0] +
							frameOrigin[0]
					][
						cellsToBeChecked[i][1] +
							currBlockOrigin[1] +
							frameOrigin[1]
					] == 1
				)
					return false;
			} else return false;
		}
		return true;
	}
	function canBlockMoveLeft(
		currBlock: string[][],
		currBlockOrigin: number[],
		frameOrigin: number[],
		board: number[][],
	) {
		let cellsToBeChecked: number[][] = [];
		for (let i = 0; i < currBlock.length; i++) {
			let j = -1;
			while (
				j + 1 < currBlock[0].length &&
				currBlock[i][j + 1] == "0"
			) {
				j++;
			}
			cellsToBeChecked.push([i, j]);
		}
		for (let i = 0; i < cellsToBeChecked.length; i++) {
			if (
				cellsToBeChecked[i][1] +
					currBlockOrigin[1] +
					frameOrigin[1] >
				-1
			) {
				if (
					board[
						cellsToBeChecked[i][0] +
							currBlockOrigin[0] +
							frameOrigin[0]
					][
						cellsToBeChecked[i][1] +
							currBlockOrigin[1] +
							frameOrigin[1]
					] == 1
				)
					return false;
			} else return false;
		}
		return true;
	}
	function handleKeyDown(event: KeyboardEvent) {
		if (
			event.key === "Enter" &&
			phase == "building" &&
			newGuess.length === 5 &&
			validGuesses.has(newGuess.toLowerCase()) &&
			!guesses.includes(newGuess.toUpperCase())
		) {
			event.preventDefault();

			newGuess = newGuess.toUpperCase();

			if (newGuess === target) {
				if (guesses.length === 0) {
					if (holdCharge == 1) {
						if (heldWord === "") {
							heldWord = target;
							target = getNewTarget();
							holdCharge = 0;
						} else {
							let temp = heldWord;
							heldWord = target;
							target = temp;
							holdCharge = 0;
						}
					}
				} else {
					phase = "falling";
					currBlock = getBlock(currentLongest);
					currBlockOrigin =
						getBoundingRect(
							currentLongest,
						)[0];
					if (currBlock.length == 0) {
						phase = "gameover";
					} else {
						rotationAnchor = [
							currBlockOrigin[0] +
								currBlock.length /
									2,
							currBlockOrigin[1] +
								currBlock[0]
									.length /
									2,
						];
					}
				}
			} else {
				if (
					!newGuess
						.split("")
						.some(
							(char, index) =>
								target[
									index
								] === char,
						)
				) {
					if (rerollCharge == 1) {
						target = getNewTarget();
						guesses = [];
						rerollCharge = 0;
					} else {
						guesses.push(newGuess);
						currBlock =
							getBlock(
								currentLongest,
							);
						currBlockOrigin =
							getBoundingRect(
								currentLongest,
							)[0];
					}
				} else {
					guesses.push(newGuess);
					currBlock = getBlock(currentLongest);
					currBlockOrigin =
						getBoundingRect(
							currentLongest,
						)[0];
				}
				if (guesses.length == maxGuesses) {
					phase = "falling";
					currBlock = getBlock(currentLongest);
					currBlockOrigin =
						getBoundingRect(
							currentLongest,
						)[0];
					if (currBlock.length == 0) {
						phase = "gameover";
					} else {
						rotationAnchor = [
							currBlockOrigin[0] +
								currBlock.length /
									2,
							currBlockOrigin[1] +
								currBlock[0]
									.length /
									2,
						];
					}
				}
			}

			newGuess = "";
		}
		if (event.key == "ArrowRight") {
			event.preventDefault();
			if (phase == "building") {
				if (
					frameOrigin[1] + 4 <
						board[0].length - 1 &&
					canFrameMoveRight(
						[
							frameOrigin,
							[
								frameOrigin[0] +
									guesses.length -
									1,
								frameOrigin[1] +
									4,
							],
						],
						board,
					)
				)
					frameOrigin[1]++;
			} else if (phase == "falling") {
				if (
					frameOrigin[1] +
						currBlockOrigin[1] +
						currBlock[0].length -
						1 <
						board[0].length - 1 &&
					canBlockMoveRight(
						currBlock,
						currBlockOrigin,
						frameOrigin,
						board,
					)
				)
					frameOrigin[1]++;
			}
		}
		if (event.key == "ArrowLeft") {
			event.preventDefault();
			if (phase == "building") {
				if (
					frameOrigin[1] > 0 &&
					canFrameMoveLeft(
						[
							frameOrigin,
							[
								frameOrigin[0] +
									guesses.length -
									1,
								frameOrigin[1] +
									4,
							],
						],
						board,
					)
				)
					frameOrigin[1]--;
			} else if (phase == "falling") {
				if (
					frameOrigin[1] + currBlockOrigin[1] >
						0 &&
					canBlockMoveLeft(
						currBlock,
						currBlockOrigin,
						frameOrigin,
						board,
					)
				)
					frameOrigin[1]--;
			}
		}
		if (event.key == "ArrowDown") {
			if (phase != "gameover") {
				event.preventDefault();
				if (
					canFrameFall(
						phase,
						frameOrigin,
						currBlockOrigin,
						guesses,
						board,
						currBlock,
					)
				) {
					frameOrigin[0]++;
				}
			}
		}
		if (event.key == "ArrowUp") {
			if (phase == "falling") {
				event.preventDefault();
				let rotated = rotateBlock(currBlock);
				let shiftedBlockOrigin = [
					Math.floor(
						rotationAnchor[0] -
							rotated.length / 2,
					),
					Math.floor(
						rotationAnchor[1] -
							rotated[0].length / 2,
					),
				];
				if (
					isValidBlockPosition(
						rotated,
						frameOrigin,
						shiftedBlockOrigin,
						board,
					)
				) {
					currBlock = rotated;
					currBlockOrigin = shiftedBlockOrigin;
				}
			}
		}
		if (event.key == "Backspace") {
			if (phase != "gameover") {
				event.preventDefault();
				newGuess = newGuess.slice(0, -1);
			}
		}
		if (event.key == " ") {
			event.preventDefault();
			if (phase == "falling") {
				while (
					canFrameFall(
						phase,
						frameOrigin,
						currBlockOrigin,
						guesses,
						board,
						currBlock,
					)
				) {
					frameOrigin[0]++;
				}
				lockBlock(
					currBlock,
					frameOrigin,
					currBlockOrigin,
				);
			}
		}
		if (newGuess.length < 5 && event.key.match(/^[a-zA-Z]$/)) {
			if (phase != "gameover") {
				newGuess += event.key;
			}
		}
	}
	function nextBuild() {
		guesses.length = 0;
		target = getNewTarget();
		newGuess = "";
		frameOrigin = [0, 2];
		currBlock = [];
		currBlockOrigin = [0, 0];
		phase = "building";
	}
	function withinFrame(frameOrigin: number[], coord: number[]) {
		if (
			coord[0] >= frameOrigin[0] &&
			coord[0] < frameOrigin[0] + 4 &&
			coord[1] >= frameOrigin[1] &&
			coord[1] < frameOrigin[1] + 5
		) {
			return true;
		} else {
			return false;
		}
	}
	function largestConnected(greens: Set<string>) {
		let longest: string[] = [];

		for (const start of greens) {
			const queue: string[] = [start];
			const visited = new Set<string>();
			const component: string[] = [];

			while (queue.length > 0) {
				const curr = queue.pop()!;

				if (visited.has(curr)) continue;

				visited.add(curr);
				component.push(curr);

				const [x, y] = curr.split(",").map(Number);

				const neighbours = [
					`${x - 1},${y}`,
					`${x + 1},${y}`,
					`${x},${y - 1}`,
					`${x},${y + 1}`,
				];

				for (const neighbour of neighbours) {
					if (
						greens.has(neighbour) &&
						!visited.has(neighbour)
					) {
						queue.push(neighbour);
					}
				}
			}

			if (component.length > longest.length) {
				longest = component;
			}
		}

		return longest;
	}
	function getBlock(currentLongest: string[]) {
		let blockSet = new Set(currentLongest);
		let boundingRect = getBoundingRect(currentLongest);
		let minX, minY, maxX, maxY;
		[[minX, minY], [maxX, maxY]] = boundingRect;
		let block: string[][] = [];
		for (let i = minX; i <= maxX; i++) {
			let currRow: string[] = [];
			for (let j = minY; j <= maxY; j++) {
				let coord = `${i},${j}`;
				if (blockSet.has(coord)) {
					currRow.push("1");
				} else {
					currRow.push("0");
				}
			}
			block.push(currRow);
		}
		return block;
	}
	function getBoundingRect(currentLongest: string[]) {
		let minX = Infinity;
		let maxX = 0;
		let minY = Infinity;
		let maxY = 0;
		for (const curr of currentLongest) {
			const [x, y] = curr.split(",").map(Number);
			if (x < minX) {
				minX = x;
			}
			if (x > maxX) {
				maxX = x;
			}
			if (y < minY) {
				minY = y;
			}
			if (y > maxY) {
				maxY = y;
			}
		}
		return [
			[minX, minY],
			[maxX, maxY],
		];
	}
</script>

<h1>LEXITRIS</h1>

<p>{target}</p>
<p>{frameOrigin}</p>
<div style="display: flex; flex-direction:column; align-items: center;">
	{#each displayGrid as boardRow, row}
		<div style="display: flex;">
			{#each boardRow as _, col}
				<div class={displayGrid[row][col].className}>
					{displayGrid[row][col].letter}
				</div>
			{/each}
		</div>
	{/each}
</div>
<svelte:window onkeydown={handleKeyDown} />

<style>
	.greenBox {
		height: 30px;
		width: 30px;
		background-color: #57c1b6;
		border: solid 1px #ffffff;
	}
	.greyBox {
		height: 30px;
		width: 30px;
		background-color: #a0a0a0;
		border: solid 1px #ffffff;
	}
	.nullBox {
		height: 30px;
		width: 30px;
		border: solid 1px #ffffff;
	}
	.graceFilledBox {
		height: 30px;
		width: 30px;
		border: solid 1px #ffffff;
	}
	.graceUnfilledBox {
		height: 30px;
		width: 30px;
		border: solid 1px #ffffff;
	}
	.tetrisFilledBox {
		height: 30px;
		width: 30px;
		background-color: #647200;
		border: solid 1px #ffffff;
	}
	.tetrisUnfilledBox {
		height: 30px;
		width: 30px;
		background-color: #64726e;
		border: solid 1px #ffffff;
	}
</style>
