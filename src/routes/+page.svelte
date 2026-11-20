<script lang="ts">
	import { onMount } from "svelte";
	type Cell = { className: string; letter: string };
	const maxGuesses = 4;

	let validGuesses: Set<string> = new Set();
	let validTargets: string[];
	let targetGen: () => number;
	const ROWS = 22;
	const COLUMNS = 9;
	let board: number[][] = $state(
		Array.from({ length: ROWS }, () => Array(COLUMNS).fill(0)),
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
	let gameMode: "NORMAL" | "ZEN" | "DAILY" = $state("NORMAL");
	const keyboardRows = ["QWERTYUIOP", "ASDFGHJKL", "ZXCVBNM"];
	const STEP_PX = 24;
	const TAP_SLOP_PX = 10;
	const TAP_MS = 250;
	const FLICK_PX = 60;
	const FLICK_MS = 250;
	let pointerStartX = 0;
	let pointerStartY = 0;
	let pointerStartTime = 0;
	let consumedX = 0;
	let consumedY = 0;
	let gestureAxis: "x" | "y" | null = null;
	let gestureActive = false;
	let controlkeys = [
		"Enter",
		"`",
		"ArrowUp",
		"ArrowLeft",
		"ArrowRight",
		"ArrowDown",
		" ",
		"Backspace",
	];
	let copied: "COPY RESULT" | "COPIED!" | "FAILED" =
		$state("COPY RESULT");
	let fallspeed: 4000 | 3500 | 3000 = $state(4000);
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
			newGuess,
		),
	);
	let score: number = $state(0);
	let wordsMade: number = $state(0);
	let totalLinesCleared: number = $state(0);

	let wordsStatus: "loading" | "ready" | "error" = $state("loading");

	async function fetchWordList(url: string): Promise<string[]> {
		const res = await fetch(url, { cache: "no-cache" });
		if (!res.ok) throw new Error(`${url}: HTTP ${res.status}`);
		const words = (await res.text())
			.split("\n")
			.map((w) => w.trim())
			.filter(Boolean);
		if (words.length === 0) throw new Error(`${url}: empty`);
		return words;
	}

	async function loadWords() {
		wordsStatus = "loading";
		const attempts = 3;
		for (let i = 0; i < attempts; i++) {
			try {
				const [nonAnswers, answers] = await Promise.all([
					fetchWordList("/guesses.txt"),
					fetchWordList("/answers.txt"),
				]);
				// Set.prototype.union needs iOS 17+, so merge manually.
				validGuesses = new Set([...nonAnswers, ...answers]);
				validTargets = answers;
				target = getNewTarget();
				wordsStatus = "ready";
				return;
			} catch (err) {
				console.error(
					`Word list load failed (attempt ${i + 1}/${attempts})`,
					err,
				);
				if (i < attempts - 1)
					await new Promise((r) =>
						setTimeout(r, 1000 * 2 ** i),
					);
			}
		}
		wordsStatus = "error";
	}

	onMount(loadWords);
	$effect(() => {
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
					if (gameMode != "ZEN") {
						frameOrigin[0]++;
					}
				} else {
					if (gameMode != "ZEN") {
						currBlock =
							getBlock(
								currentLongest,
							);
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
					if (gameMode != "ZEN") {
						frameOrigin[0]++;
					}
				} else {
					if (gameMode != "ZEN") {
						lockBlock(
							currBlock,
							frameOrigin,
							currBlockOrigin,
						);
					}
				}
			}
		}, fallspeed);
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
	async function copyHandler() {
		if (copied == "COPY RESULT") {
			try {
				await getCopyText();
			} catch {
				copied = "FAILED";

				setTimeout(() => {
					copied = "COPY RESULT";
				}, 700);
				return;
			}
			copied = "COPIED!";
			setTimeout(() => {
				copied = "COPY RESULT";
			}, 1500);
		}
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
		let linesCleared = 0;
		for (let i = 0; i < board.length; i++) {
			if (board[i].every((val) => val === 1)) {
				linesCleared++;
				for (let j = i; j > 0; j--) {
					board[j] = board[j - 1];
				}
				board[0] = Array(COLUMNS).fill(0);
			}
		}
		totalLinesCleared += linesCleared;
		score += getScoreForClear(linesCleared);

		if (score > 1000) {
			if (score > 2500) fallspeed = 3000;
			else fallspeed = 3500;
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
	function getScoreForClear(linesCleared: number) {
		const lineScore = [0, 40, 100, 300, 1200, 6000];
		return lineScore[linesCleared];
	}
	function updateGrid(
		board: number[][],
		phase: "building" | "falling" | "gameover",
		guesses: string[],
		frameOrigin: number[],
		currBlockOrigin: number[],
		target: string,
		currBlock: string[][],
		newGuess: string,
	) {
		let tempGrid: Cell[][] = Array.from({ length: ROWS }, () =>
			Array.from({ length: COLUMNS }, () => ({
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
						} else if (
							r === guesses.length &&
							newGuess[c]
						) {
							className =
								"fillingBox";
							letter = newGuess[c];
						} else if (
							r === guesses.length
						) {
							className =
								"fillingBox";
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
		if (gameMode == "DAILY") {
			const roll = targetGen();
			const targetIndex = roll * validTargets.length;
			return validTargets[
				Math.floor(targetIndex)
			].toUpperCase();
		} else {
			const roll = Math.random();
			const targetIndex = roll * validTargets.length;
			return validTargets[
				Math.floor(targetIndex)
			].toUpperCase();
		}
	}
	function mulberry32(a: number) {
		return function () {
			var t = (a += 0x6d2b79f5);
			t = Math.imul(t ^ (t >>> 15), t | 1);
			t ^= t + Math.imul(t ^ (t >>> 7), t | 61);
			return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
		};
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
			const row = guesses.length + frameOrigin[0] + 1;
			for (
				let i = frameOrigin[1];
				i < frameOrigin[1] + 5;
				i++
			) {
				if (row < ROWS) {
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
					ROWS
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
	function handleKeyDown(key: string) {
		if (
			key === "Enter" &&
			phase == "building" &&
			newGuess.length === 5 &&
			validGuesses.has(newGuess.toLowerCase()) &&
			!guesses.includes(newGuess.toUpperCase())
		) {
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
						wordsMade += 1;
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
					wordsMade += 1;
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
		if (key == "ArrowRight") {
			if (phase == "building") {
				if (
					frameOrigin[1] + 4 <
						board[0].length - 1 &&
					canFrameMoveRight(
						[
							frameOrigin,
							[
								frameOrigin[0] +
									guesses.length,
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
		if (key == "ArrowLeft") {
			if (phase == "building") {
				if (
					frameOrigin[1] > 0 &&
					canFrameMoveLeft(
						[
							frameOrigin,
							[
								frameOrigin[0] +
									guesses.length,
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
		if (key == "ArrowDown") {
			if (phase != "gameover") {
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
		if (key == "ArrowUp") {
			if (phase == "falling") {
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
		if (key == "Backspace") {
			if (phase != "gameover") {
				newGuess = newGuess.slice(0, -1);
			}
		}
		if (key == "`") {
			if (phase != "gameover") {
				reset();
			}
		}
		if (key == " ") {
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
		if (newGuess.length < 5 && key.match(/^[a-zA-Z]$/)) {
			if (phase != "gameover") {
				newGuess += key.toUpperCase();
			}
		}
	}
	function handlePointerDown(e: PointerEvent) {
		if (e.pointerType === "mouse") return;
		gestureActive = true;
		pointerStartX = e.clientX;
		pointerStartY = e.clientY;
		pointerStartTime = performance.now();
		consumedX = 0;
		consumedY = 0;
		gestureAxis = null;
		(e.currentTarget as HTMLElement).setPointerCapture(e.pointerId);
	}
	function handlePointerMove(e: PointerEvent) {
		if (!gestureActive) return;
		const dx = e.clientX - pointerStartX;
		const dy = e.clientY - pointerStartY;
		if (gestureAxis === null) {
			if (Math.abs(dx) < STEP_PX && Math.abs(dy) < STEP_PX)
				return;
			gestureAxis = Math.abs(dx) > Math.abs(dy) ? "x" : "y";
		}
		if (gestureAxis === "x") {
			const steps = Math.trunc(dx / STEP_PX) - consumedX;
			for (let i = 0; i < Math.abs(steps); i++) {
				handleKeyDown(
					steps > 0 ? "ArrowRight" : "ArrowLeft",
				);
			}
			consumedX += steps;
		} else {
			const steps = Math.trunc(dy / STEP_PX) - consumedY;
			for (let i = 0; i < steps; i++) {
				handleKeyDown("ArrowDown");
			}
			if (steps > 0) consumedY += steps;
		}
	}
	function handlePointerUp(e: PointerEvent) {
		if (!gestureActive) return;
		gestureActive = false;
		const dx = e.clientX - pointerStartX;
		const dy = e.clientY - pointerStartY;
		const elapsed = performance.now() - pointerStartTime;
		if (
			Math.abs(dx) < TAP_SLOP_PX &&
			Math.abs(dy) < TAP_SLOP_PX &&
			elapsed < TAP_MS
		) {
			handleKeyDown("ArrowUp");
		} else if (
			dy > FLICK_PX &&
			dy > Math.abs(dx) &&
			elapsed < FLICK_MS
		) {
			handleKeyDown(" ");
		}
		gestureAxis = null;
	}
	function handlePointerCancel() {
		gestureActive = false;
		gestureAxis = null;
		consumedX = 0;
		consumedY = 0;
	}
	function nextBuild() {
		guesses.length = 0;
		target = getNewTarget();
		newGuess = "";
		frameOrigin = [0, 2];
		currBlock = [];
		currBlockOrigin = [0, 0];
		phase = "building";
		rerollCharge = 1;
		holdCharge = 1;
	}
	function getDateString() {
		const pad = (n: Number) => String(n).padStart(2, "0");
		const d = new Date();

		const s =
			pad(d.getDate()) +
			pad(d.getMonth() + 1) +
			d.getFullYear();
		return s;
	}
	function reset() {
		if (gameMode == "DAILY") {
			const s = getDateString();
			targetGen = mulberry32(Number(s));
		}
		nextBuild();
		board = Array.from({ length: ROWS }, () =>
			Array(COLUMNS).fill(0),
		);
		score = 0;
		wordsMade = 0;
		totalLinesCleared = 0;
		heldWord = "";
		fallspeed = 4000;
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
	function chooseMode(mode: "NORMAL" | "ZEN" | "DAILY") {
		gameMode = mode;
	}

	function getCopyText() {
		let copyText = "Lexitris" + "\n" + gameMode;
		let s = getDateString();
		if (gameMode == "DAILY") {
			copyText += " #" + s + "\n";
		}
		copyText += "\nSCORE = " + score + "\n";
		copyText += "WORDS MADE = " + wordsMade + "\n";
		copyText += "LINES CLEARED = " + totalLinesCleared + "\n";

		return navigator.clipboard.writeText(copyText);
	}
</script>

<div
	class="gameArea"
	style:--rows={displayGrid.length}
	style:--cols={displayGrid[0]?.length ?? 1}
	role="application"
	aria-label="Lexitris board"
	onpointerdown={handlePointerDown}
	onpointermove={handlePointerMove}
	onpointerup={handlePointerUp}
	onpointercancel={handlePointerCancel}
	style="display: grid; grid-template-rows:auto auto 1fr auto;  height: 95dvh;"
>
	<div style="font-size:30px ; font-weight:bold; margin: 15px;">
		LEXITRIS
	</div>
	<div style="font-size:25px ; font-weight:bold;">
		<button
			onclick={() => {
				chooseMode("DAILY");
				reset();
			}}>DAILY</button
		>
		<button
			onclick={() => {
				chooseMode("NORMAL");
				reset();
			}}>NORMAL</button
		>
		<button
			onclick={() => {
				chooseMode("ZEN");
				reset();
			}}>ZEN</button
		>
	</div>

	{#if phase == "gameover"}
		<div class="endPanel">
			<div
				style="background-color: #d7e837; font-weight:bold;"
			>
				<div
					style="font-size:25px ; font-weight:bold; "
				>
					GAME OVER
				</div>
				<br />
				<div
					style="font-size:25px ; font-weight:bold; "
				>
					SCORE
				</div>
				<div
					style="font-size:25px ; font-weight:bold; "
				>
					{score}
				</div>
				<div
					style="font-size:25px ; font-weight:bold; "
				>
					WORDS MADE
				</div>
				<div
					style="font-size:25px ; font-weight:bold; "
				>
					{wordsMade}
				</div>
				<div
					style="font-size:25px ; font-weight:bold; "
				>
					LINES CLEARED
				</div>
				<div
					style="font-size:25px ; font-weight:bold; "
				>
					{totalLinesCleared}
				</div>
			</div>
			<button onclick={copyHandler}>{copied}</button>
			<button onclick={reset}>RESTART</button>
		</div>
	{:else}
		<div class="gameGrid">
			<div class="stats">
				<p
					style="font-weight:bold; text-align: center;"
				>
					SCORE
				</p>
				<p
					style="font-weight:bold; text-align: center;"
				>
					{score}
				</p>
				<br />
				<br />
				<p style="text-align: center;">On Hold</p>
				<p style="text-align: center;">
					<b>{heldWord}</b>
				</p>
				<br />
				<br />
				<p
					class={rerollCharge == 1
						? "activeCharge"
						: "inactiveCharge"}
				>
					REROLL
				</p>
				<p
					class={holdCharge == 1
						? "activeCharge"
						: "inactiveCharge"}
				>
					HOLD
				</p>
			</div>
			<div
				style="display: flex; flex-direction:column; align-items: center;"
			>
				<p
					style="font-size:25px; font-weight:bold;background-color: #d7e837;"
				>
					{target}
				</p>
				{#each displayGrid as boardRow, row}
					<div style="display: flex;">
						{#each boardRow as _, col}
							<div
								class="cell {displayGrid[
									row
								][col]
									.className}"
							>
								{displayGrid[
									row
								][col].letter}
							</div>
						{/each}
					</div>
				{/each}
			</div>
		</div>
		<div class="keyboard">
			{#each keyboardRows as row, i}
				<div class="keyboardRow">
					{#if i == 2}
						<button
							class="key wideKey"
							type="button"
							onclick={() =>
								handleKeyDown(
									"Enter",
								)}>ENTER</button
						>
					{/if}
					{#each row.split("") as letter}
						<button
							class="key"
							type="button"
							onclick={() =>
								handleKeyDown(
									letter,
								)}
							>{letter}</button
						>
					{/each}
					{#if i == 2}
						<button
							class="key wideKey"
							type="button"
							onclick={() =>
								handleKeyDown(
									"Backspace",
								)}>DEL</button
						>
					{/if}
				</div>
			{/each}
		</div>
	{/if}
</div>
{#if wordsStatus === "error"}
	<div class="wordsError" role="alert">
		Couldn't load the word list, so guesses can't be checked.
		<button type="button" onclick={loadWords}>RETRY</button>
	</div>
{/if}
<svelte:window
	onkeydown={(e) => {
		if (controlkeys.includes(e.key)) e.preventDefault();
		handleKeyDown(e.key);
	}}
/>

<style>
	:global(body) {
		font-family: "Alexandria", helvetica, sans-serif;
	}
	.gameArea {
		touch-action: none;
		user-select: none;
		--cell-height: min(
			calc((95dvh - 290px) / var(--rows)),
			calc((100vw - 160px) / var(--cols)),
			25px
		);
		--cell-width: var(--cell-height);
		--cell-border: solid 1px #ffffff;
		--cell-text-align: center;
		--cell-line-height: var(--cell-height);
	}
	.cell {
		box-sizing: border-box;
		height: var(--cell-height);
		width: var(--cell-width);
		border: var(--cell-border);
		text-align: var(--cell-text-align);
		line-height: var(--cell-line-height);
		font-size: calc(var(--cell-height) * 0.6);
	}
	.greenBox {
		background-color: #d7e837;
	}
	.greyBox {
		background-color: #a0a0a0;
	}
	.nullBox {
		background-color: #f4f8ff;
	}
	.fillingBox {
		background-color: #ffd8fe;
	}
	.tetrisFilledBox {
		background-color: #4067c4;
	}
	.tetrisUnfilledBox {
		background-color: #e5edff;
	}
	.graceFilledBox {
	}
	.graceUnfilledBox {
	}
	.activeCharge {
		background-color: #d7e837;
		width: 80px;
		border: solid 2px white;
		text-align: var(--cell-text-align);
		line-height: var(--cell-line-height);
		border-radius: 3px;
		outline: solid 2px #d7e837;
	}
	.inactiveCharge {
		background-color: #a0a0a0;
		width: 80px;
		border: solid 2px white;
		text-align: var(--cell-text-align);
		line-height: var(--cell-line-height);
		border-radius: 3px;
		outline: solid 2px #a0a0a0;
	}
	.gameGrid {
		display: grid;
		grid-template-columns: auto 1fr;
		align-items: center;
	}
	.stats {
		padding-right: 14px;
	}
	.endPanel {
		padding-left: 14px;
	}
	.keyboard {
		display: none;
		flex-direction: column;
		align-items: center;
		gap: 6px;
		margin-top: 12px;
	}
	.keyboardRow {
		display: flex;
		gap: 4px;
	}
	.key {
		min-width: 28px;
		height: 40px;
		padding: 0 4px;
		font-size: 14px;
		font-weight: bold;
		font-family: inherit;
		border: solid 1px #a0a0a0;
		border-radius: 4px;
		background-color: #ffffff;
		touch-action: manipulation;
	}
	.wordsError {
		position: fixed;
		top: 12px;
		left: 50%;
		transform: translateX(-50%);
		width: max-content;
		max-width: calc(100vw - 32px);
		padding: 10px 14px;
		display: flex;
		gap: 12px;
		align-items: center;
		background-color: #ffe1e1;
		border: solid 1px #c44040;
		border-radius: 6px;
		font-size: 14px;
		z-index: 10;
	}
	.wordsError button {
		font-family: inherit;
		font-weight: bold;
		touch-action: manipulation;
	}
	.key:active {
		background-color: #d7e837;
	}
	.wideKey {
		min-width: 44px;
		font-size: 11px;
	}
	@media (pointer: coarse) {
		.keyboard {
			display: flex;
		}
	}
</style>
