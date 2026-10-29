## Ticket 1:

### Actual cause

**File / function:** `src/screens/MenuScreen/MenuScreenController.ts`, `loadStatsFromServer()`

**What was wrong:** The client was trying to get the endpoint`/api/stat`, but the server only has the route `/api/stats`. The response was a 404,  and we got the  `"Unable to load stats"` on the main screen instead of the stats.

### Fix

Changed the URL in `loadStatsFromServer()` from `"/api/stat"` to `"/api/stats"` 

### How did you find it?

#### Hypothesis testing

**What hypothesis ultimately led you to identify the buggy code, and why?**

My hypothesis was that the stats request itself was failing, not the display code. "Stats unavailable" is only shown when `updateStats()` receives `null`,. I used command f to find this error, and then was able to traceback that it was not calling the right api endpoint.

#### Backward tracing

1. **Symptom:** The menu screen showed "Stats unavailable", and the console showed `Error: Unable to load stats`.
2. **Traced back to:** `MenuScreenView.updateStats()`, which shows "Stats unavailable" only when `stats === null`. 
3. **Traced back to:**  The Network tab showed the request to `/api/stat` returning an error (404) instead of JSON.
4. **Buggy code:** `src/screens/MenuScreen/MenuScreenController.ts`, `loadStatsFromServer()`, `fetch("/api/stat")`. The server's route is `/api/stats`.

### Confirming the fix

After changing the URL and reloading, the menu screen showed the Best / Average / Scores saved line instead of "Stats unavailable".

---

## Ticket 2:

### Actual cause

**File / function:** `server/index.ts`, the `GET /api/stats` route handler

**What was wrong:** The loop that finds the best score used the wrong comparison: `if (entries[i].score < best)`. The server always returned `best: 0`, so the menu screen showed "Best: 0" even when the leaderboard had higher scores.

### Fix

Changed the comparison to `if (entries[i].score > best)`, so `best` is replaced whenever a higher score is found. 

### How did you find it?

#### Hypothesis testing

**What hypothesis ultimately led you to identify the buggy code, and why?**

My hypothesis was that the server was calculating `best` incorrectly. The Network tab showed we were getting response but the value was incorrect.

#### Backward tracing

1. **Symptom:** The start/menu screen showed "Best: 0", even though the leaderboard had scores above 0.
2. **Traced back to:** `MenuScreenView.updateStats()`
3. **Traced back to:** The `/api/stats` response in the Network tab, which contained `"best": 0`. So the wrong value came from the server.
4. **Buggy code:** `server/index.ts`, the `app.get("/api/stats", ...)` handler, the line `if (entries[i].score < best)`, which should be `>`.

### Confirming the fix

After fixing the comparison and restarting the server, the `/api/stats` response in the Network tab showed `best` equal to the highest score on the leaderboard, and the menu screen showed the correct "Best:" value.

## Ticket 3:

### Actual cause

**File / function:** `src/screens/GameScreen/GameScreenView.ts`, `randomizeLemonPosition()`

**What was wrong:** The parentheses were in the wrong place:

Every time the lemon was clicked, it moved to one of the four corners of the padded play area instead of a random spot.

### Fix

Moved the closing parenthesis so the random decimal is scaled by the range *before* rounding:

### How did you find it?

#### Hypothesis testing

**What hypothesis ultimately led you to identify the buggy code, and why?**

The lemon was moving after each click, just never to anywhere new. My hypothesis was that the click handling worked but the random number generation was broken. 

#### Backward tracing

1. **Symptom:** After each click, the lemon jumped to one of the four corners of the play area and never to anywhere in between.
2. **Traced back to:** `this.view.randomizeLemonPosition()` The click handler was running correctly, so the problem was the logic.
3. **Traced back to:** how `randomX` and `randomY` were calculated.
4. **Buggy code:**  `Math.round(Math.random()) * (maxX - minX) + minX`. `Math.round(Math.random())` is always 0 or 1, so each coordinate could only be its min or max.

### Confirming the fix

After moving the parentheses and reloading, I clicked the lemon many times. It landed in different spots across the whole play.

## Ticket 4:

### Actual cause

**File / function:** `src/screens/GameScreen/GameScreenController.ts`, `endGame()`

**What was wrong:** `endGame()` wrote `this.stopTimer;` without parentheses so that means it is never called.

It kept on calling endGame as time decreased.

- It  replayed `gameOverSound` each time, so the sound played every second.
- Clicking Play Again showed the menu, but jumped back to "GAME OVER!".

### Fix

Changed `this.stopTimer;` to `this.stopTimer();`. `stopTimer()` now actually runs, calling `clearInterval(this.gameTimer)` and setting `gameTimer` to `null`. The countdown stops the moment the game ends, so `endGame()` runs exactly once per game.

### How did you find it?

#### Hypothesis testing

**What hypothesis ultimately led you to identify the buggy code, and why?**

Sound kepy on replaying so I thought something with the endgame was buggy. I checked where the timer is supposed to be cleared, in `endGame()`, and found that `stopTimer` was referenced but never called.

#### Backward tracing

1. **Symptom:** After the game ended, the game-over sound replayed every second.
2. **Traced back to:** `ResultsScreenController.showResultsFromServer()`. This function both submits the score with a POST to `/api/leaderboard` and plays `gameOverSound`
3. **Traced back to:** `GameScreenController.endGame()`, which switches to the results screen. Once the timer reached 0 the interval was still running and repeating.
4. **Buggy code:** `src/screens/GameScreen/GameScreenController.ts`, `endGame()`

### Confirming the fix

After the fix, I ensured play again returned to the menu and stayed there. After clicking Start, the new game ran for the full time.

## Ticket 5:

### Actual cause

**File / function:** `src/screens/ResultsScreen/ResultsScreenModel.ts`, the `LeaderboardEntry` type, and `src/screens/ResultsScreen/ResultsScreenView.ts`, `updateLeaderboard()`

**What was wrong:** The client and server used different names for the date field. The server saved each entry with a time field, but the client expected a timestamp field. When the client tried to display entry.timestamp, it got undefined because that field didn't exist in the server's response. 

### Fix

Renamed the field from `timestamp` to `time` in the client's `LeaderboardEntry` type, and changed `updateLeaderboard()` to read `${entry.time}`. The client now reads the property the server actually sends. 

### How did you find it?

#### Hypothesis testing

**What hypothesis ultimately led you to identify the buggy code, and why?**

The score showed correctly, but the date always showed "undefined". In JavaScript, a template string prints "undefined" when a property does not exist. My hypothesis was that the client was reading the date under a different property name than the one the server sends. 

#### Backward tracing

1. **Symptom:** The "Top Scores" list showed `1. 18 - undefined`. The score was right, but the date was missing.
2. **Traced back to:** `ResultsScreenView.updateLeaderboard()`, which builds each line as ``${index + 1}. ${entry.score} - ${entry.timestamp}``. 
3. **Traced back to:** `ResultsScreenController.submitScoreToServer()`, which gets the entries from `POST /api/leaderboard`. 
4. **Buggy code:** `src/screens/ResultsScreen/ResultsScreenModel.ts`,  `timestamp: string;`, and the matching read at `ResultsScreenView.ts:106`. Both use `timestamp`, while `server/index.ts` stores and sends the field as `time`.

### Confirming the fix

After the fix, I played a game and checked the Game Over screen. Each "Top Scores" entry showed the score and the date.
