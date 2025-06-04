# March-Madness-Optimizer

## THE PROBLEM:
Every spring, college basketball teams compete for the national championship in a single-elimination, 64-team tournament called March Madness.  Many people join pools where participants predict the winner of all of the games (bracket). The participants that best predict the outcome of the tournament (based on the pool’s scoring system) capture a fixed share of the prize fund. The task is to submit a single bracket that maximizes expected payout against N unknown opponents.

## WHY THE PROBLEM IS INTERESTING:
Most casual players approach a March Madness pool by attempting to pick the most likely bracket. That strategy is adequate in a small pool, where simply being “more accurate than a handful of friends” is often enough to win money. Scale changes everything. In a pool with hundreds or thousands of entries, a high-probability champion like a No. 1 seed appears on many brackets. Even if that favorite wins the tournament, you must still beat a crowded field that shares your champion. This is especially problematic if the No. 1 seed is overselected and appears as a winner on a higher percentage of brackets than the team’s actual odds of winning the tournament. In effect, the most likely outcome becomes less profitable. The response is to pick a “dark-horse” which has good odds of winning the tournament but is selected by only a few participants. But how contrarian is too contrarian? Pick an upset that is so unlikely it almost never lands, and your expected return collapses; stay too close to consensus, and brackets with over-selected picks dramatically reduce your expected return. With statistics and simulation, we can find near optimal solutions.

## HIGH-LEVEL APPROACH:
1. Simulate the world: generate realistic samples of both complete tournament outcomes and expected opponents’ brackets.  
2. Optimize: explore the 63-pick decision space to find the bracket with the highest mean payout across all simulated pools.

---

## GENERATING OPPONENT BRACKETS:
Using data from CBS Sports, which reports the percentage of users selecting each team in every round, I randomly sampled brackets from these empirical distributions. This produced realistic pools of N opponents.

## GENERATING REALISTIC TOURNAMENTS:
Lacking a proprietary edge over market forecasts, I relied on KenPom advanced statistics (points scored minus points allowed per 100 possessions adjusted for opponent, possessions per game adjusted for opponent) to build a pairwise win-probability model. Running 10,000 full-bracket simulations yielded aggregate outcome frequencies that matched betting-market odds almost exactly, validating the approach.

## DATASET FOR THE OPTIMIZER:
I chose to make 10,000 instances of bracket pools. Each bracket pool instance contains a tournament outcome and N opponent brackets. I chose to generate 10,000 pool instances, which is a large enough number to represent the probability space while being small enough to compute quickly. To calculate my bracket’s payout in a pool, I would need to score my bracket, score opponent brackets, and calculate my payout based on my placement among opponent brackets.

For proof of concept, I started out with a naive prototype, using nested Python loops: for each simulated tournament, generate N opponent brackets and score each one. Unfortunately, evaluating 10,000 pools this way would have taken days of compute time—far too slow.

To speed things up, I turned to 3-D tensor operations in NumPy and precomputed the score every opponent bracket would achieve against every tournament outcome. I represented a single tournament outcome by a vector of length 63, where the first 32 entries represent the winners of the first round. Each entry contained the integer of the team that won that game. The next 16 entries represented the second round, with each entry containing the integer of the team that won that game. The remaining rounds were also represented the same way. I also used the same representation to generate 10,000 opponent brackets. 

Using this representation along with tensor operations, I efficiently computed a 10,000 x 10,000 output matrix, where entry i,j represented the score achieved by opponent bracket i on the tournament outcome j. Since the optimization step only needed to compare my bracket score to opponent scores, I only needed to save the scores and not the actual entries for the opponent brackets. 

To create outcomes to optimize against, I then randomly sampled from this matrix. I created 10,000 samples where each sample contained a tournament outcome and N opponent scores for that outcome. This process took less than a minute, far better than the multi-day compute time required with the naive approach. Furthermore, since my bracket could only make revenue if it placed in the top K, I discarded all but the top K scores for each outcome. This again cut down on memory costs and improved efficiency in the optimizer.

## OPTIMIZER:
I designed a custom optimizer that takes advantage of the tournament bracket’s structure. To do the optimization I started with an initial guess chosen from the best of the opponent brackets. Since later rounds are worth significantly more points than earlier rounds, the optimizer worked backwards attempting to pick the winner of the championship game first, and then picking winners of the Final Four games, to the Elite Eight games, and so on. At each game, I looked at the teams that could possibly reach and win that game. To keep the computation manageable, I didn’t evaluate every possibility. Instead, I focused only on the top-rated teams among the eligible ones. For each candidate team, I temporarily updated the bracket to make that team win the given game (and all preceding games), recalculated my expected payout, and kept the change only if it improved the expected payout. Repeating this process for several passes led the bracket to converge to a final solution in just a few minutes.

## RESULT:
The optimizer produced the bracket with an extremely high expected payout against the empirical distribution of opponent strategies—effectively giving me a near optimal entry for my pool. In simulations, this bracket was in the money 8% of the time, far higher than the 1.3% you would expect from a naively chosen bracket.

I submitted this bracket, named BracketBot, to a 150-person pool with a $2,000 prize. By the Final Four, BracketBot had correctly predicted every Elite Eight team and all Final Four teams. It led the pool in total points and all of its remaining picks were favored by betting markets. And most importantly, this bracket was the leader of all 36 brackets that had the same finals winner, validating the approach. 

Anecdotally, with strong odds of winning, I decided to hedge (place offsetting bets) using DraftKings to lock in value. Although BracketBot ultimately lost the pool due to Duke’s unexpected defeat, my hedging strategy paid off—I walked away with $530 in winnings.
