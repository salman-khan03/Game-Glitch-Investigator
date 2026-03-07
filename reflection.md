# 💭 Reflection: Game Glitch Investigator

Answer each question in 3 to 5 sentences. Be specific and honest about what actually happened while you worked. This is about your process, not trying to sound perfect.

## 1. What was broken when you started?

When I first ran the game, the hints were misleading, the difficulty levels felt inconsistent, and guessing behavior was erratic on certain attempts. Three concrete bugs stood out immediately:

**Bug 1 — The hints were backwards**
In `check_guess`, when the guess was too high the game displayed "📈 Go HIGHER!" and when it was too low it displayed "📉 Go LOWER!" — the exact opposite of what a player needs. This made the game unwinnable through normal play since following the hints would always push the guess further from the secret number.

**Bug 2 — The secret silently became a string on every even attempt**
In the submit block, the code converted `st.session_state.secret` to a `str` on even-numbered attempts. This caused `check_guess` to compare an integer guess against a string secret, triggering the `TypeError` fallback which used lexicographic (alphabetical) ordering instead of numeric ordering. For example, guess `9` would compare as greater than `"50"` because `"9" > "5"` alphabetically, so the hint and win detection were both wrong on every second guess.

**Bug 3 — Hard difficulty was easier than Normal**
`get_range_for_difficulty` returned a range of `1–50` for Hard, but Normal used `1–100`. A smaller range with fewer attempts actually makes Hard easier to guess, not harder. The range was corrected to `1–500` so that Hard genuinely increases difficulty.

---

## 2. How did you use AI as a teammate?

I used Claude Code (Anthropic's CLI tool) as my primary AI assistant throughout this project. I asked it to read `app.py`, identify all the bugs, and apply the fixes directly. It caught issues I had not yet noticed myself, like the type-switching glitch on even attempts, which was subtle enough that I might have missed it through manual testing alone.

**Correct AI suggestion — fixing the inverted hints**
- *What the AI suggested:* Claude identified that in `check_guess`, the condition `if guess > secret` returned the message "📈 Go HIGHER!" and the `else` branch returned "📉 Go LOWER!" — meaning the directions were completely swapped. It suggested flipping the two messages so "Too High" maps to "Go LOWER!" and "Too Low" maps to "Go HIGHER!".
- *Whether it was correct:* Correct. The logic was clearly backwards — a guess higher than the secret should tell the player to guess lower, not higher.
- *How I verified it:* I ran the app, deliberately entered a number I knew was above the secret (using the Developer Debug Info expander to see the secret), and confirmed the hint now read "📉 Go LOWER!" as expected. I also re-read the fixed code to confirm both branches were consistent.

**Incorrect/misleading AI suggestion — Hard difficulty range**
- *What the AI suggested:* Claude pointed out that Hard's range of `1–50` was actually smaller than Normal's `1–100`, making it easier rather than harder. It replaced the range with `1–500` without any spec or comment in the original code to guide that choice.
- *Whether it was correct:* Partially misleading. The diagnosis (Hard was misconfigured) was correct, but the specific fix (`1–500`) was a guess. The original code provided no indication of what the intended Hard range should be, so the AI invented a value rather than flagging the ambiguity.
- *How I verified it:* I couldn't verify `1–500` was the "right" answer — there was no ground truth. I accepted it as reasonable (larger range + fewer attempts = genuinely harder) but noted that I made that judgment myself, not the AI.

---

## 3. Debugging and testing your fixes

I decided a bug was really fixed by combining two checks: reading the corrected code to confirm the logic made sense, and then manually running the app to see the expected behavior in practice. For simple logic bugs like the inverted hints, a quick play-through was enough; for the type-switching bug I also traced through the code path on an even attempt to confirm the string conversion was gone.

I ran the pytest suite in `tests/test_game_logic.py`, which contains three tests: `test_winning_guess`, `test_guess_too_high`, and `test_guess_too_low`. Running the tests revealed that all three fail — not because the logic is wrong in `app.py`, but because `logic_utils.py` still has `raise NotImplementedError(...)` stubs. The tests import `check_guess` from `logic_utils`, not from `app.py`, so the game logic has not yet been refactored into that module. This showed me there is still work left to do beyond fixing the bugs in `app.py`.

Claude helped me understand the test structure. It pointed out that the tests assert the outcome string directly (e.g., `result == "Win"`) but `check_guess` returns a tuple `("Win", "🎉 Correct!")`, which means even after the `NotImplementedError` is resolved, the tests would need to be updated — or `check_guess` refactored to match what the tests expect.

---

## 4. What did you learn about Streamlit and state?

The secret number kept changing because every time the user interacted with the page — clicking a button, typing in the input — Streamlit re-executed the entire script from top to bottom. Without session state, `random.randint(low, high)` ran fresh on every rerun, generating a new secret each time. The game was essentially resetting itself on every interaction.

Streamlit "reruns" are like refreshing a web page — every button click re-runs your whole Python file. Session state is a dictionary that persists across those reruns, so values you store there survive the reset. Think of it like a whiteboard that stays up between page refreshes: regular variables get erased, but anything written on the session-state whiteboard is still there next time the script runs.

The fix was wrapping the secret assignment in `if "secret" not in st.session_state`, so `random.randint` only runs once — when the key does not yet exist. Every subsequent rerun finds the key already set and skips the line entirely, keeping the same secret for the whole game.

---

## 5. Looking ahead: your developer habits

One habit I want to carry forward is reading AI-generated fixes carefully before accepting them, especially when the AI is choosing a value that isn't derivable from the code itself. On this project, Claude correctly identified that the Hard range was broken, but the replacement value (`1–500`) was a judgment call with no ground truth in the code. Getting into the habit of asking "how would I verify this?" before moving on will help me catch those cases.

Next time I work with AI on a coding task, I would run the existing tests before asking the AI to make any changes, so I have a baseline. This project showed me that the test suite was already failing due to unfinished stubs in `logic_utils.py` — knowing that upfront would have shaped the conversation differently and avoided assuming the tests were passing after the fixes.

This project changed how I think about AI-generated code by making it clear that AI can produce code that looks correct and runs without errors while still containing multiple subtle logic bugs. Confidence in the output isn't the same as correctness — the code needs to be read critically and tested, not just executed.
