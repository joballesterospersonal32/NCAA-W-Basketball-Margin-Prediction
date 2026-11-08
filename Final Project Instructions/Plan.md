Yes. I can read the CSVs and the parquet files. `team_box.parquet` opens: 120,032 rows, two per game, and it matches `teams.parquet` on game, team, score, and field-goal attempts. The dictionary is the SportsDataverse pipeline overview (play-by-play, team box, player box, and poll snapshots). It does not list these columns. The revised plan is saved in `Final Project Instructions/Plan.md`.

The rankings file is one poll. `rankings.csv` has 80 data rows, all dated 6 April 2026, covering the AP Top 25 and the Coaches Poll, including receiving-votes rows and a few duplicated teams. Joining that file onto earlier games would paste an April 2026 poll onto games that had not been played. The rank that changes during the season is `home_current_rank` and `away_current_rank` on `games.parquet`. It is empty for 2017–2023 and filled for 2024–2026, with 99 meaning unranked. UCLA in 2026 moves from 3 in November to 1 in April, so the column is not the final poll copied onto every game. Stage 2 uses that column. The rankings file is only the spot check for the last week.

`game_box` is the player box score. Play-by-play stays out. The target stays one row per game: home score minus away score.

## Plan

**1. Freeze the sample**

Purpose: drop incomplete games and name the files the paper can be rebuilt from.

Inputs: `games.parquet` for the schedule and `team_box.parquet` for the box score, seasons 2017–2026. Keep a game when it is completed, both scores are present, the two scores differ, and both teams have a box-score row. Basketball has no ties, so equal scores are recording errors. In this extract, 18 kept games, all on 7 January 2017, are recorded as 0–0 in both `games.parquet` and `team_box.parquet` even though their box scores show shot attempts. Dropping them leaves 54,002 of 55,154 scheduled games. Elo burn-in also reads completed games from 2006–2016, scores only.

Output: a sample table. Rows are seasons. Columns are scheduled games, games with a box score, games dropped for equal scores, postseason games, and neutral-site games. Note that 2021 has 3,823 games and 2026 has 6,054.

**2. Build the one-row game table**

Purpose: stop the two team-rows of a game from being treated as two games.

Inputs: the step 1 sample. Target is home score minus away score, which is never 0 after step 1. `home_court` is 1 only when the site is not neutral. `postseason` is 1 when `season_type` is 3. Carry the ids, the date, the season, and both rank fields.

Output: the modeling table. Later steps use this grain only.

**3. Define the same-game Four Factors**

Purpose: turn the team box into Oliver’s four factors and the extra stats for the penalized models.

Inputs: the paired home and away rows from `team_box.parquet`. For each team,

- shooting: (field goals made + 0.5 × three-pointers made) / field goals attempted
- turnovers: turnovers / (field goals attempted + 0.44 × free throws attempted + turnovers)
- rebounding: offensive rebounds / (offensive rebounds + opponent defensive rebounds)
- free throws: free throws attempted / field goals attempted
- possessions: field goals attempted − offensive rebounds + turnovers + 0.44 × free throws attempted

Stage 1 uses the four home-minus-away differences. The expanded matrix adds home-minus-away differences in assists, steals, blocks, and fouls per possession, and the possessions difference. Drop a game if either team has a zero denominator.

Output: the four formulas in the text, and a column dictionary in the appendix.

**4. Fit the Stage 1 least-squares model**

Purpose: measure how much of the same-game margin the Four Factors account for. The factors are built from the shots that produce the score, so a high R² is an accounting association.

Inputs: margin and the four differences, seasons 2017–2026. Standardize the four predictors. Leave the margin in points.

Output: a paper table with R², adjusted R², RMSE, MAE, and one coefficient per factor in points of margin per standard deviation. Shooting is expected to be the largest. Oliver’s 40/25/20/15 split is a directional reference, not a target the coefficients must hit. The appendix repeats R² by season.

**5. Fit ridge, lasso, the LARS path, and elastic net**

Purpose: see which of assists, steals, blocks, fouls, and possessions the lasso retains once the Four Factors are present. With this many games and fewer than ten predictors, the penalty is expected to move the fit very little.

Inputs: the expanded matrix, standardized inside each training fold. Five-fold cross-validation on RMSE, split by game. Keep the largest penalty within one standard error of the best mean RMSE. Elastic net mixing weights are 0.25, 0.5, and 0.75.

Output: a LARS-path figure with the chosen lasso penalty marked, and a table of cross-validated RMSE plus the variables the lasso keeps. Full coefficients go in the appendix.

**6. Attach the pre-game poll rank**

Purpose: use the rank going into the game, and keep the April 2026 poll from being applied to earlier seasons.

Inputs: `home_current_rank` and `away_current_rank`. Rank points are 26 minus the rank when the rank is 1 through 25, and 0 when the rank is 99. `d_rank` is the home value minus the away value. `rank_known` is 1 for 2024–2026 and 0 for 2017–2023. When the rank is unknown, `d_rank` is 0. The AP rows of `rankings.csv` are used only for the spot check. Duplicate receiving-votes rows are dropped first. That file is not joined onto games.

Output: an appendix spot-check of games in the week of 6 April 2026 against the AP poll. A last-week mismatch is reported. It does not remove the feature, because the rank moves across the season. The paper says poll rank is observed for 2024–2026 only.

**7. Build the Elo rating**

Purpose: score opponent strength. Beating a strong team by a wide margin moves a team more than beating a weak team by a little. Past average margin does not. Carrying part of last season’s rating is what the first games of a new season are missing.

Inputs: every completed game from 2006 through 2026 with two different scores, scores and site only, in date order. Games recorded with equal scores are skipped: 70 such games across those seasons, including the 18 removed in step 1. Games on the same date all read the rating from the start of that date, then update after the date.

- Each team starts at 1500 the first time it appears.
- Pre-game home win probability is \(1 / (1 + 10^{-(R_h - R_a + H) / 400})\). \(H\) is a home-court term in rating points, and it is 0 on a neutral site. It is not stored in the team rating.
- After the game, a team moves by \(K \times m \times (S - P)\). \(S\) is 1 for a win and 0 for a loss. \(m = \ln(|\mathrm{margin}| + 1) \times 2.2 / (0.001 \times D + 2.2)\), where \(D\) is the winner’s pre-game rating advantage.
- Before the next season’s first game, replace rating \(R\) with \(1500 + \rho (R - 1500)\).

Choose \(K\) from {20, 32}, \(H\) from {0, 60, 100}, and \(\rho\) from {0, 0.5, 0.75} by the expanding-window RMSE of the Elo baseline, validation seasons 2018–2024 only. Then lock them. Seasons 2006–2016 are burn-in, so 2017 does not open with every team at 1500.

Output: pre-game `d_elo`, home rating minus away rating, on every modeling row. An appendix figure shows two teams across 2025–26. The chosen \(K\), \(H\), and \(\rho\) go in the text.

**8. Build the season-to-date box-score features**

Purpose: add the Four Factors and past margin from earlier games only.

Inputs: each team’s completed games from the same season with a strictly earlier date. Averages restart each season. Past margin and the four past factors are exponentially weighted means (EWM) of those games. The weight on the game \(k\) games back (\(k = 1\) is the most recent) is \(\lambda^{k-1}\), where \(\lambda = 0.5^{1/h}\) and \(h\) is the half-life in games. Dividing by the sum of the weights keeps it a proper average. In pandas this is `ewm(halflife=h, adjust=True).mean().shift(1)`, grouped by team and season. The same \(h\) applies to past margin and to all four factors. \(h = \infty\) gives equal weights, which is the plain season-to-date mean; pandas does not accept an infinite half-life, so that case is `expanding().mean().shift(1)`. Past factors are averaged over earlier games with a valid box score. If a team has two games on one date, both read the value from before that date, because `shift(1)` alone would let the second game see the first. \(h\) is chosen in step 12.

A game enters the common Stage 2 set only when both teams have at least five earlier games with scores and five with a valid box score. Those are plain counts of earlier games, kept separate from the EWM weights.

Date field. Steps 7 and 8 use one column for order and for "strictly earlier": `game_date`, read from `games.parquet` and matched to `game_date` on `team_box.parquet`. It is the schedule's calendar date, already a date with no clock. The `date` column is a UTC timestamp. On 46,793 games its UTC day is the next calendar day, so a late tipoff would be assigned to the wrong day (a timestamp of 2006-04-05T00:30Z is `game_date` 2006-04-04). `date`, and the UTC day of `game_date_time`, are not used for order or for the leakage rule.

Perturbation test. Pick a game G on date D. Change its home and away scores and its box-score values, then rebuild every pre-game feature: the EWM past margin, the EWM past factors, `d_elo`, and the five-game counts. The test passes when no feature changes for any game on a date earlier than D, or on D itself, for any team. That also shows that same-date games do not see each other. As a positive control, features must change for later games of the two teams in G. If nothing changes, the test is broken. Run it on several games drawn at random across seasons, including at least one from the first week of a season and one from 2026. Save the test as a script in the code submission so it can be rerun.

Features on that set: `home_court`, `postseason`, past-margin difference, four past-factor differences, `d_elo`, `d_rank`, and `rank_known`.

Output: a table of how many games each season survive the five-game rule, and a feature audit listing the latest date each column is allowed to use. Each EWM feature uses only the team’s games before the game date. The audit has one row naming `game_date` as the date field and stating that the perturbation test passed.

**9. Fit four baselines**

Purpose: separate home court, raw past margin, and opponent-adjusted strength. The fitted past-margin model is the fair comparator for Elo: both have a slope and a home-court term, so the only difference is the strength measure, EWM past margin versus `d_elo`. Elo is the baseline the other models have to beat.

Inputs: the common set.

- Home court alone, fit by least squares.
- Past margin, no fitted terms: home team’s EWM past margin minus the away team’s, using the locked \(h\).
- Past margin + home court, fitted: \(\text{margin} = b_0 + b_1 \times (\text{past-margin difference}) + b_2 \times \text{home\_court}\), fit by least squares on the training seasons only. It uses the EWM past margin with the locked \(h\), the common set, and the same expanding-window scheme as the Elo baseline.
- Elo: least squares of margin on `d_elo` and `home_court`, slope fit on the training seasons only.

The appendix also scores the locked Elo rule on games that fail the five-game rule, which is where a carried-over rating has something to do and a season-to-date average does not.

Output: four rows in the final test table, plus the early-season Elo row in the appendix. Before the 2026 test, on the 2024 validation season, fit both past-margin versions and Elo on 2017–2023 only and report how much of Elo’s gap over the no-fitted-terms version the fitted version closes. That check does not use 2025 or 2026.

**10. Fit linear, ridge, and lasso**

Purpose: test whether the pre-game factors and the poll rank add anything once Elo and home court are already in the model.

Inputs: the step 8 features. The past-margin and past-factor features are EWMs with the locked \(h\). Ridge and lasso standardize the continuous predictors from the training seasons only. Penalties are chosen in step 12.

Output: 2026 coefficients in the appendix, including the features the lasso keeps. Test error goes in the step 13 table.

**11. Fit the additive spline, the local regression, and the random forest**

Purpose: see whether a bend in the strength relationship beats the linear Elo model.

Inputs: the same features and training seasons. The past-margin and past-factor features are EWMs with the locked \(h\).

- Splines on past-margin difference, past effective-field-goal difference, `d_elo`, and `d_rank`. The other terms stay linear. Effective degrees of freedom is the trace of that smoother.
- Local linear regression on `d_elo` and the past-margin difference. Report the bandwidth. Leave it off the degrees-of-freedom plot.
- A random forest on the full feature set, fixed seed, scored only on later seasons. Out-of-bag error draws in future games from the same season, so it is not a result in this project.

Output: spline degrees of freedom and forest depth in the tuning appendix. Test errors in the step 13 table.

**12. Tune through 2024, check 2025, lock, then test 2026**

Purpose: choose every setting without seeing the test season.

Inputs: expanding windows. Train on 2017 and score 2018, then train through each next season and score the following one, up through scoring 2024. Average validation RMSE decides. Near-ties keep the simpler setting. The Elo triple is chosen first and frozen.

The half-life \(h\) is chosen second, from {5, 10, 20, \(\infty\)}. \(\infty\) is the plain season-to-date mean and is the default. Score each \(h\) by the expanding-window validation RMSE over 2018–2024, using the step 10 least-squares model on all step 8 features, which has no penalty of its own to tune. On a near-tie, keep the larger \(h\). Lock \(h\) before the 2025 check and the 2026 test, and do not tune it on 2025 or 2026. The other models are then tuned on the frozen Elo rating and the locked \(h\).

Refit on 2017–2024 and score 2025 once. Do not retune. Refit the same settings on 2017–2025 and score 2026 once. A 2026 rating includes 2025 results because those games are over. It does not include the 2026 game being predicted.

Output: an appendix grid. One sentence in the paper on the chosen settings and the 2025 check.

**13. Report the 2026 test**

Purpose: answer the forecast question on games the tuning never used.

Inputs: 2026 rows of the common set. RMSE and MAE in points of home margin.

Output: the main table. Rows are home court, past margin (no fitted terms), past margin + home court (fitted), Elo, linear, ridge, lasso, additive spline, local regression, and random forest.

**14. Bootstrap the test errors**

Purpose: separate a real gap from the size of the 2026 sample.

Inputs: the step 13 predictions, held fixed. Resample 2026 games 1,000 times. Recompute each model’s RMSE and its gap versus Elo on the same games.

Output: 95% percentile intervals on the test table. A model is described as better than Elo only when the interval for its RMSE gap lies entirely below zero. Also show the gap of Elo versus past margin with no fitted terms, and the gap of Elo versus the fitted past-margin model. The intervals hold the fitted models fixed, so they are test-set noise, not the variability from refitting or from re-running Elo.

**15. Plot validation error against effective degrees of freedom**

Purpose: show the bias-variance tradeoff for the two models that have a smoother trace.

Inputs: the ridge path and the spline path from step 12. Mark the chosen penalty and the 2026 RMSE at that degrees of freedom.

Output: one figure, validation RMSE against effective degrees of freedom. Elo’s \(K\), the forest, and the local regression are judged in the test table.

**16. Write the result sentences**

Purpose: say what you expected, what you found, and what follows, as associations.

Expectations to write against: Stage 1 R² is high and shooting is the largest standardized coefficient. The lasso keeps the Four Factors. Elo beats the fitted past-margin model on 2026. The gap between Elo and that fitted model is the value of opponent adjustment and carry-over. The gap between the two past-margin rows is the cost of not fitting a slope and a home term. Another Stage 2 model counts as an improvement only when its RMSE gap versus Elo lies entirely below zero. Poll rank is reported for 2024–2026.

Output: four or five conclusion sentences. The abstract is then one paragraph of about four to six sentences.

## Report requirements

Related work still has to be written: Oliver’s definitions and the 40/25/20/15 weights, Elo (1978) and the margin-of-victory update in step 7, one NCAA or women’s basketball prediction paper, and the course sources for ridge, lasso, LARS, penalized splines, and random forests. That is about one to one and a half pages.

The methods section states, in one sentence, that recent games are weighted by an exponential decay with the chosen half-life \(h\), and that \(h\) was chosen on 2018–2024 validation.

A leakage test was run: after changing one game's scores and box score, pre-game features are unchanged for every game on that date and every earlier date, and they do change for later games of those two teams.

The paper itself holds the sample table, the Stage 1 coefficient table, the LARS figure, the 2026 table with intervals, and the degrees-of-freedom figure. The rest goes in the appendix. Submit the code and the one-row table with the PDF. The box score in the code is `team_box.parquet`.

## Gaps

- Confirmed: there is no week-by-week AP poll for 2017–2023. The only poll file is the 6 April 2026 snapshot. Stage 2 rank is `home_current_rank` and `away_current_rank`, and that field starts in 2024. The rankings file is the last-week spot check and is not joined onto earlier games.
- `teams.parquet` is a copy of the team box, and `game_box` is the player file.
- The Box link in the proposal and these SportsDataverse files still need one sentence naming which extract was fit.
- The bootstrap does not refit the models or re-run Elo.
- Stage 1 will look strong because the factors use the same shots as the score. The conclusion says that before it interprets Stage 2.

This is the plan to build.