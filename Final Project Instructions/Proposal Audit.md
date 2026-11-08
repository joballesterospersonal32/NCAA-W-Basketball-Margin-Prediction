Proposal audit for STAT 57900
Approved individual proposal, Women's NCAA basketball scoring margin. Scored against the final-project handout. Models are not built yet, so this scores the plan.
3
Met
5
Partially met
1
Missing
Build after the literature list and the feature rules are written down
The question and the two-stage design are accepted. The gap that would hurt the report is the missing literature review, followed by five design choices the proposal leaves open. Locking those choices is what makes the models defensible.
Requirement by requirement
Source: Final Project Instructions and Submission and Formatting, applied to Final_Project_Proposal.md. Data checks use games.csv (114,705 games, seasons 2006–2026).

| Requirement                                 | Status        | Why                                                                                                                                                                                                                         |
| ------------------------------------------- | ------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Tied to statistical machine learning        | Met           | Ridge, lasso, elastic net, splines, random forest, chronological cross-validation, and effective degrees of freedom are core course material. Applying them to a dataset is an allowed project type.                        |
| Individual submission                       | Met           | You are working alone. That is within the allowed size, so no extra justification is needed.                                                                                                                                |
| The problem                                 | Met           | Two questions, one target: how much of scoring margin the Four Factors explain in the same game, and how well margin can be predicted from information available before tipoff.                                             |
| Existing techniques and a literature review | Missing       | Dean Oliver is named. The proposal has no papers, no citations, and no account of existing forecasting methods. The report is required to contain this section.                                                             |
| Methodology and improvements                | Partially met | The two stages, the pre-tipoff rule, and the baselines are the right design. Formulas, the model list, rank missingness, and the one-row-per-game construction are not fixed.                                               |
| Numerical experiments                       | Partially met | Expanding-window tuning, a chronological holdout, RMSE, MAE, bootstrap intervals, and a degrees-of-freedom plot are the right skeleton. Train and test seasons, the rolling window, and the bootstrap unit are unspecified. |
| Meaning of the results                      | Partially met | You already wrote what you expect: the Four Factors should explain most of the same-game margin, and regularization should help little in Stage 1. The plan does not yet say what implication would follow from Stage 2.    |
| Data the instructor can reproduce           | Partially met | A public Box link is cited. The files on disk are SportsDataverse wehoop extracts, and a Box link can expire. The report has to name the files actually used.                                                               |
| Fits the 5–8 page report                    | Partially met | The method list in the abstract is wider than an eight-page paper. Stage 2 against the baselines has to stay the main result. Tuning grids belong in an appendix.                                                           |

Lock these before fitting models
Stage 1, same game
One row per game. Target is home score minus away score. Four Factors from Oliver's formulas, with possessions estimated and opponent rebounds taken from the other team's box-score row.
Fit ordinary least squares on the Four Factors, then the same model plus assists, steals, blocks, fouls, and possessions. Use the lasso path to see which extra stats remain. Report ridge and elastic net as a short comparison.
Stage 2, before tipoff
Predictors are each team's rolling averages from earlier games that season, plus rank entering the game. Same-game box scores stay out of this stage.
Baselines first: home court alone, and the difference in the two teams' past average margin. Then linear, one penalized linear model, additive splines, and a random forest. Hold out the most recent complete season. Judge models by RMSE and MAE.
Facts from the files that the proposal should absorb
114,705 games, 2006–2026
2026 has 6,054 games
2021 has 3,823 games
Home rank filled on 15.4% of games
9,329 neutral-site games
Season 2026 is a full season in this file, so it can be the chronological holdout. Season 2021 is short and should be mentioned. Most teams are unranked, so rank needs an unranked indicator. Neutral-site games need their own indicator and should not receive a home-court bonus.
games.csv includes home_current_rank and away_current_rank. Confirm that value is the rank entering that game before using it. team_box.csv has the counting stats the Four Factors need. teams.csv has the same header and opening row as team_box.csv, so it is not a team directory. game_box.csv is a player box score and is outside the approved question.
Final PDF rules
These apply to the report, not to the proposal. Submit one PDF on Brightspace, 5–8 pages plus references, with the code and a usable copy of the data. Use the course template: about 10-point Times, Type 1 fonts, a single-paragraph abstract of roughly 4–6 sentences, figure captions under the figure, table captions above the table, and complete references in chronological order when several are cited together.
The abstract draft in the proposal is longer than that and still has placeholders. Cut it to one paragraph once the two result sentences exist.