# Capstone acceptance criteria

## 1. Every run finishes in under 10 seconds, including the first ask after a quiet period

Target: every run has total_seconds under 10.0 in logs/runs.jsonl, across 3 cold first asks (each after at least 15 minutes with no FitFindr run, started in a fresh Terminal window) and 10 warm asks, all with AI201_CACHE=0 and the query 'vintage graphic tee under $30'. That is 13 of 13.

Why this target: My warm runs sit around 2 to 3 seconds and the one idle cold run was 4.1 seconds, so 10 seconds still clears ordinary Gemini jitter and a slightly slower first ask, but a 180-second hang would miss. Three cold runs is enough to see whether the first ask stays under the limit without turning the eval into an hour of waiting.

## 2. A bad API key shows up as an error record in the log

Target: with GEMINI_API_KEY=invalid and AI201_CACHE=0, running the same vintage-tee ask makes the process exit non-zero, and the new log line has status "error", error_type "ModelUnavailable", a non-empty error_message, and session_error null. 3 of 3 tries.

Why this target: This is the failure I can force on purpose without depending on caption text. One try is not enough to trust the finally write; five feels like overkill once I have seen the exception type. Three is enough to catch a logger that swallows the failure or marks it ok.

## 3. An empty query through run_agent lands as handled_error

Target: calling run_agent("", ...) directly (not python app.py ask '', which drops into interactive mode) makes the new log line have status "handled_error", a non-empty session_error, null error_type and error_message, and null stage_seconds for suggest_outfit and create_fit_card. 5 of 5 tries.

Why this target: No model is involved; search returns [] and the branch stops. If it holds once it should hold every time, so 5 of 5. I score run_agent itself because the CLI empty-string path never reaches the agent.

## 4. Two overlapping requests to serve.py run one after the other

Target: with the server started by python serve.py, send two POST /ask requests with the vintage-tee query at the same moment. The second waits while the first finishes (the second run's start, which is its timestamp minus its total_seconds, is no earlier than the first run's timestamp, within 0.1 seconds), and both get a log line with status "ok". 3 of 3 pairs.

Why this target: One pair is not enough to trust the lock. Five pairs adds setup effort without more information, since the lock should behave the same way every time.

## 5. Cache-off token use stays in a measurable band

Target: over ten warm cache-off runs of the vintage-tee query, the mean of prompt_tokens + output_tokens in the log is between 500 and 1000.

Why this target: I measured about 737 tokens per run (roughly 587 in, 150 out), which is about $0.00055 at the list prices in the README. The band fails if counting drops to near zero or if prompts blow up past 1000, without freezing the mean at exactly 737. Scoring the mean over ten matches how I computed cost; a single-run cap would live or die on one chatty reply. I set this band after I had already measured the mean, so it can only fail if token counting breaks.
