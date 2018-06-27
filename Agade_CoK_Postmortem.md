# Agade Code Royale Postmortem

I will describe my AI which finished 2nd on the Code of Kutulu competition on Codingame.

## Introduction

My AI was based on a Paranoid (Minimax) search algorithm. I did not believe a straight up Paranoid would be the best on this contest. I figured a non-exhaustive search would be required to get much needed depth. Indeed the game rules are a bit complex which favors search algorithms over heuristics. Search depth is very important to see damage coming, but the game being a simultaneous multiplayer one, you have to search to depth 8 just to see 2 turns into the future (4 player, 4 moves per turn). But I started my AI as Paranoid and did not achieve/ran out of time to find a better search algorithm.

As in Code Royale I did not take the time to produce a local arena, which would have been very useful.

## Paranoid

Paranoid is the multiplayer (>2 players) equivalent of Minimax where you assume that you are playing against an opponent who controls several explorers. As in Minimax, in Paranoid you have a player maximising his score and the other player minimising it. However this is very pessimistic in the multiplayer setting because not only are you assuming that your opponents are out to get you in particular, but you are also assuming that they will collaborate against you. A more reasonable assumption is that every player will selfishly maximise his score, pretty much ignoring other players. Indeed, in a multiplayer game, why should player 0 spend energy hurting player 1, it would only help all other players. The minimax equivalent of this more reasonable assumption is called MaxN. While in Paranoid, during the search you back up a score which one player maximises and the others minimise, in MaxN you back up an array of scores and each player maximises his. However, since the Paranoid algorithm has a more direct correspondence to 1v1 Minimax, you can use Alpha-Beta Pruning to increase search depth whereas MaxN has much more limited [options](https://ijcai.org/Proceedings/03/Papers/098.pdf) for pruning. While in 1v1, Alpha-Beta Pruning will, in the best theoretical case, double you search depth, in Paranoid this generalises to raising your tree size to the power of `(N-1)/N` where N is the number of players. In our case this is the equivalent of increasing search depth by 33% (in the best case). As with Minimax the best-case pruning is attained by trying the best moves first in the search leading to the topic of [move ordering](https://chessprogramming.wikispaces.com/Move+Ordering). As a reference, [here](https://webdocs.cs.ualberta.ca/~nathanst/papers/comparison_algorithms.pdf) is an article I read in my Tron days on MaxN and Paranoid.

Another question for search algorithms in this game is how to handle the simultaneous move aspect. A traditional implementation of minimax here is equivalent to having the last player know what the other players will play this turn. In lack of a better solution I just let my opponents have that advantage in my search, making the Paranoid search even more pessimistic.

## Performance and depth

For move ordering I used, in order of priority the [hash move](https://chessprogramming.wikispaces.com/Hash+Move), the [killer heuristic](https://chessprogramming.wikispaces.com/Killer+Move) (two slots) and the [history heuristic](https://chessprogramming.wikispaces.com/History+Heuristic) (index by player id, position of player and action). For the hash of the best move table I used an idea from Neumann as in this case it would be complicated to perform [Zobrist hashing](https://chessprogramming.wikispaces.com/Zobrist+Hashing) on the state. His idea was to have, like in Zobrist hashing, a random number for every `(action,player,depth)` tuple, start the state at a hash of 0, and XOR in every move performed. In effect not having a hash representing the state but a "delta hash" representing the difference between the state and the starting one. This has drawbacks, the hash table can no longer be thought of as a [transposition table](https://chessprogramming.wikispaces.com/Transposition%20Table) because identical states may have different hashes if different actions led to them, also the table has to be wiped every turn because the hashes are only valid with respect to the previous turn's starting state (unless you kept track of every opponent move during the game and updated the delta hash accordingly?). 

I measured that out of a theoretical best of `^0.75` I was pruning my search tree to around `^0.8`. Tested and optimised locally on game states from real games converted to a string format.

I could perform at least 20000 simulations per turn, as a worst case, around 40000 was common with peaks around 80000 (actually like 170000 on some approximately empty states). Depending on the map and the position of the players (less branching in corridors) I could reach depth 2-3 with 4 players, 3-4 with 3 players. With 2 players I don't remember but according to Magus' [cgstats](http://cgstats.magusgeek.com/app/code-of-kutulu/Agade) I would finish 1st 50% more often than second and third (the numbers have changed now because CG purges game history). So I would describe the search depth in 1v1 and 1v2 as, acceptable.

## Simulating LIGHT effect

Many chose to leave the minion-repulsion effect of LIGHT out of their simulation due to the computational cost of running A* searches for every minion. Instead of diregarding it completely I had a fast lower bound calculation of the LIGHT's repulsion effect. The idea of the lower bound is that a light, say a distance of 3 away, will cover your feet and all cells within a distance of 2 from you.

```
constexpr int Light_Distance_Bonus{4};//From game rules
constexpr int Light_Range{5};//From game rules
int real_distance_weighted(const player &p,const minion &e){
	const int dist_p_e{real_distance(p,e)};//Get minion-player distance from distance cache
	int light_repulsion{0};
	for(light &l:L){
		const int dist_light_p{real_distance(p,l)};
		light_repulsion+=(min(max(0,Light_Range+1-dist_light_p),dist_p_e))*Light_Distance_Bonus;//+1 to light range because it is 5 + square you stand on
	}
	return dist_p_e+light_repulsion;
}
```

Hopefully I did not make mistakes here. There are probably better approximations that could be made here by thinking geometrically about the 3 points (player,light,minion) and the known distances between them.

## Eval

* `1e10-depth*1e9` if state is a victory. Devalue search depth in order to promote winning faster.
* `-1e10+depth*1e9` if state is a loss (I didn't distinguish 2nd, 3rd and 4th place). Value search depth to promote losing later.
* `0+depth*0.1` if state is a draw (all players left died on same turn). Value depth to draw later. If enemies make mistakes you may win instead.
* `+my_sanity`
* `-enemy_sanity` for every enemy
* `-2*max(0,dist_to_nearest_explorer-Loneliness_Distance)` where Loneliness_Distance=2 from the game rules
* `-0.05*dist_to_explorer` for every explorer
* `+dist_to_nearest_minion`
* `+0.025*distance_to_minion` for every minion
* `+0.015*distance_to_minion_target` for every minion. So a minion targeting the guy right next to me is still bad according to this criterion.
* `-19*players_I_have_yelled_at`
* `-0.35*slashers_targetting_me`
* `-0.05*wanderers_targetting_me`

All distances mentioned are "real" distances as found by a BFS (shortest path length). I really did not make much effort on the eval and especially its coefficients. When I tried touching the eval the effect on the ranking was unnoticeable and I figured I should concentrate on improving search depth.

For the coefficient on enemy sanity I had a bit of a dilemma. If you were doing a MaxN search you would need your evals to be constant sum, so the sum of each player's eval should add to some constant ("What's good for me must be bad for all other players"). In that case the eval should be `my_sanity-sum_of_enemy_sanity/3` (or any eval divided by the sum of all evals). Indeed without the `1/3` coefficient a MaxN search would prefer a state where sanity is 9 for me and 10 for enemies (eval of 9-30) over a state where sanity is 21 for me and 20 for enemies (eval of 21-60). All this to say it kind of makes sense to have a `1/3` coefficient on enemy sanity. But I was worried that in my paranoid search enemies would then perceive their health as less valuable and be willing to sacrifice it to hurt me, which real players would probably not do.

## Allowed moves

I only allowed my explorer to YELL and LIGHT (if at least one enemy is further than 7 moves away) in the search. Ranking seemed poorer when allowing enemy explorers to yell (and adapting the eval for it) for some reason. I did not attempt allowing enemies to LIGHT. PLAN was not allowed for any explorers, I had an override of WAIT moves to use PLAN when my sanity was below 200. I thought of having a better heuristic for the use of PLAN, but in real games by the time my sanity was 200 I was already in a group of players so I didn't think about this too much. Not allowing moves obviously also improves search depth.

## Things I tried

* I experimented with MaxN, perhaps too briefly and found it quite poor, perhaps due to the abysmal search depth, not always reaching depth 2.
* I tried using a non-exhaustive simulated annealing search: Start a solution vector where all players wait, then briefly optimise players one by one. I figured this may converge to a Nash equilibrium (a situation where no player has incentive to deviate from his current plan). Perhaps I should have insisted on this solution but my implementation played poorly. Or maybe the idea is fundamentally flawed as the search may oscillate instead of converging.
* Starting the Paranoid search from one of the enemies to make it less pessimistic, i.e: not all enemies would know what move I was going to make that turn. Perhaps I should have insisted with this.

## Things I didn't try

* Disregarding players and minions past a certain distance in the search.

## Things to improve

* Performance. I actually had some nice performance improvements the last Monday morning but the ranking was so unclear I resubmitted my last known version 15s from the end. Glad I didn't get the "no code in IDE" bug.
* I should have looked at more replays but those I did I noticed a recurring pattern of my AI staying at the back of the group, becoming a perfect target for a YELL. I was hoping to fix this by just allowing enemies to YELL but I didn't manage.
* Win rate in 1v3. My AI had an equal probability of finishing first or last, 2nd and 3rd being rarer. Perhaps Paranoid is really inadequate for the 1v3 part of the game.

## Conclusion

I finished with 1100 lines, including all testing code. I took an embarrassingly long time to debug my engine, finding out wanderers were either spawning twice too fast or even despawning in my engine on Sunday. Should have debugged using other players' next turn position as Robostac mentioned.

I could have improved performance and a few other things with more time, perhaps squeezing out a win against blasterpoard (what a terrible name) but I feel like he played better than me, especially considering his lower computation time, so I'm happy to just congratulate him.



