

# My FPL Team Generator


## TLDR

I wrote an FPL team generator. Its suggestion for the optimal team from gw18-gw28 is:

-   Pope
-   Cancelo - Dias - Walker-Peters - Bednarek
-   De Bruyne - Fernandes - Rashford - Grealish - El Ghazi
-   Martial

Subs:

-   Rodrigo - Brewster - Hause

(Starters and subs would change depending on fixtures. I ran the
algorithm and picked the team a few days ago. The output if you run
the algorithm now is a bit different, but I kept this team becuase
I already made the transfers and wanted the team on this post to be
the same as my actual team)

[Github repo here](https://github.com/dghosef/FPL-team-generator)


## Intro

This is my third season of FPL and ever since I was introduced to the
game, I've been meaning to try my hand at writing an automated team
picker. This year, thanks to COVID, Stanford gave us an abnormally
long break. This, coupled with the fact that I can't really go
anywhere(also thanks to COVID) has given me a perfect opportunity to
do just that. After a lot of tinkering, I'm at the point where I'm
happy enough with the algorithm's output for me to share it, though
it's far from perfect. I do plan to improve it more to see how good it
can get, and hopefully by next season I will have ironed out many of
the kinks. First, credit where credit is due

-   [This project by Reddit user u/blubbersassafras greatly inspired my algorithm](https://www.reddit.com/r/FantasyPL/comments/dg1to7/an_analysis_of_overanalysis_my_adventure_in_fpl)
-   [Excellent guide for linear programming with Python/PuLP for FPL](https://medium.com/@joseph.m.oconnor.88/linearly-optimising-fantasy-premier-league-teams-3b76e9694877)
-   [Tutorial for accessing the FPL API with python that helped quite a bit](https://medium.com/@conalldalydev/how-to-get-fantasy-premier-league-data-using-python-f99f50ab0da)
    
    Warning: The rest of this post constains a lot of technical
    details. I don't blame you if you want to skip the rest. However,
    the math isn't that complex(lots of multiplication and division)
    and the programming section isn't that long and is entirely
    skippable.
    
    Also, I don't have much of a math/stats background, so forgive me
    if you notice any mistakes. I'd appreciate it if you commented
    any mistakes or suggestions, either in the code, conceptually, or
    in my explanation so I can improve this for next season.


## Algorithm overview

1.  Player Points Prediction

    In order to pick the best team, the algorithm first tries to
    predict each player's point count over the next 10 gameweeks.
    
    The general process for player point prediction is:
    
    1.  Calculate each team's relative attacking and defensive strength and from that predict the score of each game
    2.  Calculate how involved each player is in their team's assists and goals as a percentage
    3.  From step 1 and 2, predict the number of goals/assists of each player
    4.  If the player is a gk, def, or mid, calculate probability of keeping a cs in each game
    5.  If the player is a gk or def, calculate the probability of conceeding 2+ goals
    6.  If the player is a gk, try to predict number of saves
    7.  From the above data, try to predict bonus points
    8.  Convert the predicted goal values, assist values, etc into a predicted fpl score
    
    All player data was taken from the FPL api.
    
    The first and most important step of the algorithm is to find the
    relative attacking and defensive strength of each team(where a
    higher attacking strength and lower defensive strength is
    better). Again, a lot of the credit for this part goes to
    u/blubbersassafras for his writeup on a similar process(see
    above). In order to find the relative strength of any given team,
    the program loops through that team's past few scores and
    compares how many goals that team conceeded and scored compared
    to how the average team would have fared. For example, let's say
    team A beat team B 2-1 where team B is a fairly average team and
    we want to find team A's attacking and defensive strength. In
    order to find team A's relative attacking strength from this
    game, the program compares the number of goals team A scored to
    the number of goals the average team would score. Since team B is
    a fairly average team and the average team conceeds 1.36
    goals/game, team A outperformed the average by 2/1.36, so its
    attacking strength is 2/1.36. However, if team B had a defense
    that was half as good as the average team(d. strength = 2), you
    would expect the average team to score 1.36 \* 2 or 2.72
    goals. Since team A only scored 2 goals, team A only has an
    attacking strength of roughly 2/2.72. Likewise, the defensive
    strength of team A is 1/1.36 since the average team would conceed
    1.36 goals/game against team B's average attack but team A only
    conceeded 1(recall that lower defensive strengths and higher
    attacking strengths indicate a stronger team). If team B had an
    attack that was half as good as average(a. strength = 0.5), the
    average team would be expected to conceed 0.5 \* 1.36 or .68
    goals. Therefore, team A's defensive strength in this scenario
    would be 1 / 0.68 which is worse than average.
    
    Each team's attacking and defensive strength is initially set to
    equal 1(the average) and this calculation is performed for each
    team's last few games and averaged to find a total attacking and
    defensive score. This process is then repeated, this time using
    the previously calculated strengths in order to get a
    better picture of the opponent's level when assessing each
    score. And then it's repeated again using the new strengths from
    that calculation. And so on and so forth. I included some
    python-esque pseudocode that shows the process in a bit more
    detail.
    
	```python
        def team_strengths():
        	 team_list = ['ARS', 'SOU', 'WBA', ...]
        	 # Past scores in the form:
        	 # {Team: [(Opposition1, Team goals, Opposition goals), ...] ... }
        	 scores = {'ARS': [('CRY', 2, 0), ('SOU', 1, 3), ('BUR', 1, 0), ...],
        			   'CRY': [('ARS', 0, 2), ('MUN', 3, 3), ('WBA', 4, 0) ...]
        			   ...}
        	 # Initialize attacking and defensive strengths to 1 for each team
        	 d_strengths = {'ARS': 1, 'BUR': 1, 'SOU': 1 ...}
        	 a_strengths = {'ARS': 1, 'BUR': 1, 'SOU': 1 ...}
        	 avg_goals = 1.36 # epl average goals/team/game
        	 num_iterations = 50
        	 for i in range(num_iterations): # Update strengths num_iterations times
        		  # The new strengths calculated from this iteration
        		  new_d_strengths = dict()
        		  new_a_strengths = dict()
        		  # find strengths for each team
        		  for current_team in team_list:
        			   current_team_d_strengths = list()
        			   current_team_a_strengths = list()
        			   for score in scores[current_team]:
        					opposition = score[0] # opposition team
        					goals_for = score[1] # current_team goals scored
        					goals_against = score[2] # current_team goals conceeded
        					opposition_d_strength = d_strengths[opposition][0]
        					opposition_a_strength = a_strengths[opposition][0]
        					# Find defensive strength from this game. Lower is better
        					current_team_d_strengths.append(goals_against / avg_scored / opp_a_strength)
        					# Find attacking strength from this game. Higher is better
        					current_team_a_strengths.append(goals_for / avg_scored / opp_d_strength)
        			   # Average the newly calculated strengths to find the team's
        			   # overall strength
        			   new_d_strengths[current_team] = average(current_team_d_strengths)
        			   new_a_strengths[current_team] = average(current_team_a_strengths)
        		  # If we haven't completed num_iterations, redo the process
        		  # newly calculated team strengths
        		  a_strengths = new_a_strengths
        		  d_strengths = new_d_strengths
        	 return d_strengths, a_strengths
	```python
    
    The initial idea was that attacking and defensive strengths would
    converge after enough iterations. In reality, they often
    alternated between 2-4 values. As a result, the final algorithm
    calculates team strengths for 54 iterations and uses the average
    of the last 4. It is possible that a higher iteration count would
    have lead to a convergence, but I'm not sure how long that would
    take and the average is likely very close to the theoretical
    value of convergence. In my actual implementation, I max out the
    defensive and offensive strength of each game to be 3 so as to
    remove the impact of outliers like WBA scoring against MCI who
    hadn't conceeded in a month and the 7-2 AVL-LIV game. Also, in my
    implementation, the past 10 gameweeks are considered where the 5
    most recent fixtures are weighted twice as heavily as the other 5
    fixtures. See the team<sub>strengths</sub> function at the top of
    [predict<sub>points.py</sub>](https://github.com/dghosef/FPL-team-generator/blob/main/src/predict_points.py) for the actual implementation. Once we have
    each team's strength, we can predict any score. The formula for
    the score is
    
    avg<sub>goals</sub> = 1.36
    (team1 goals, team2 goals) = (avg<sub>goals</sub> \* team1<sub>a</sub><sub>strength</sub> \* team2<sub>d</sub><sub>strength</sub>, avg<sub>goals</sub> \* team2<sub>a</sub><sub>strength</sub> \* team1<sub>d</sub><sub>strength</sub>)
    
    The rest of point prediction is fairly straightforward. As many
    of you know, FPL provides their own custom statistics entitled
    creativity and threat, where 100 creativity roughly corresponds
    to 1 assist and 100 threat roughly corresponds to 1 goal. [Here](https://www.facebook.com/832058563619842/posts/how-ict-influence-creativity-threat-index-is-calculatedinfluence-evaluates-the-d/1113864845439211/) is
    a link to a breakdown of how the calculation works(the linked
    post doesn't cite any sources so I'm not completely sure how
    accurate it is, though it shouldn't be too hard to verify by
    comparing the actual creativity/threat values to what the formula
    in the post predicts they would be). In order to calculate the
    number of goals a player is going to score, we calculate that
    player's threat per minute(tpm) and their team's overall threat
    per minute over the last 10 games. Then we figure out the
    expected goals their team is going to score in a game using the
    score formula above and apply the formula
    
    predicted<sub>goals</sub><sub>player</sub> = (player<sub>tpm</sub>/team<sub>tpm</sub>) \* predicted<sub>goals</sub><sub>team</sub>
    
    We do the same process to figure out assists but with creativity
    per minute(cpm) so that
    
    expected assists = (player<sub>cpm</sub>/team<sub>cpm</sub>) \* expected<sub>goals</sub> \* .75
    
    I multiply by .75 because I'd estimate roughly 25% of goals don't
    have an assist. 
    
    I won't go into too much detail for the rest of the steps, but
    here is a quick overview of the highlights
    
    -   We can use the poisson distribution formula to get probability of cleansheet = e<sup>(-predicted goals conceeded)</sup>
    -   The poisson distribution formula is also used to find probability of opposition scoring 2+ goals(calcualte the probability of each score and then add up the probabilities of each score where the opposition scores 2+ goals)
    -   Goalkeeper save count is predicted by looking at their saves/opposition attacking strength/game over the last 10 games and then multiplying by new opposition attacking strength
    -   Bonus points are calculated by dividing the number of bps of a player(found using the [official bps formula and the previously calculated stats](https://www.premierleague.com/news/106533)) by 16
        -   I chose 16 because the average number of predicted bonus points when I use 16 is quite close to the real life average
    -   Points are found using multiplication. 2 points are added for playing time
        -   For example, if Ings, has 1.2 predicted goals and 0.23 predicted assists, his point value is 1.2\*4 + .23\*3 + 2
    -   Players who have played less than 240 mins over the last 4 games have their point value overriden to 0 as they aren't considered a regular starter
    -   Players who are marked as 25%, 50%, and 75% have their point values multiplied accordingly(see the [get<sub>data</sub> function](https://github.com/dghosef/FPL-team-generator/blob/main/src/pick_team.py))
    -   Players who are suspended and have long term injuries have their point values modified accordingly

2.  Team selection

    The idea of the team generation algorithm is simple: pick the
    team that maximizes predicted points(see previous section) given
    the constraints of FPL(11 starters, 3 subs, max of 3 players per
    team, budget, etc). Thankfully, python has a very convenient
    package called [PuLP](https://pypi.org/project/PuLP/) that makes solving problems like these a
    breeze. Essentially, all I had to do was input the constraints
    and predicted points and the package would spit out the optimal
    team. To pick the initial team, I inputted all of the FPL
    constraints into a PuLP model along with each player's total
    predicted points over the next 10 gameweeks. In order to take
    into account the fact that subs don't play all that much and
    captains get double points, I got the model to pick an overall
    starting XI, 3 subs, and 2 captains for the next 10 weeks with
    the assumptions that the first, second, and third subs play 20%,
    10%, and 5% of the time and each captain gets 1.5x their original
    predicted points over the 10 weeks since we rotate captains. The
    sub percents and number of captains were all picked somewhat
    arbitrarily/by experimentation and intuition rather than having
    any real statistical basis, something that could be improved in
    the future. To pick the starting XI for any given gameweek, the
    algorithm picks the starting 11 that maximizes predicted points
    for that gameweek and then orders the subs from highest predicted
    points to the lowest. The transfer process is quite similar to
    the team selection process. The only difference is I added the
    constraint that the new team must have only 1 different player
    than the original team(this can be altered to 2 players, 3
    players, etc).


## Limitations/things to be improved/modifications

-   Currently the algorithm can't determine how many transfers to make so we just make 1 every week
-   The 0.75 assists/goal is very arbitrary. In the future, I could try to find the assist to goal ratio for each team and use that instead
-   When calculating creativity and threat influences for each player, the algorithm doesn't take into account the difficulty of the teams that player played.
    -   For example, if Lingard creates a bunch of chances vs WBA and then gets benched for the hard games, he would have an unfairly high creativity influence.
    -   Could be fixed by comparing each player's creativity per minute to his team's cpm only in the games he was involved in.

-   The bonus point algorithm doesn't account for bps magnets/repellants. It also assumes bonus points scales linearly with bps which isn't that accurate.
-   There's probably a better way to guess who is going to start and or maybe estimate a probability of each player starting.
-   It probably wouldn't take too much effort to modify the team selection to make an anti-fantasy-premier-league team which would be an interesting experiment
-   I also plan to modify the algorithm to build a hivemind team which tries to optimize the percent selected amount to build the most popular team(I'm sure some has already done this though) as a comparison
-   Probably a lot more that I didn't think of or meant to remember but forgot(I had a lot of moments where I was like "oh I can improve xyz" while working on this that I'm sure I forgot)

 If you want to track my progress, I have an fpl team(team id 7742703) that I plan
to keep updating with the algorithm's choices as the season passes. It
currently is on negative points because I took a lot of hits while
experimenting. My hope is to iron out the kinks for next season.  I
left out a number of details because this post is already sort of
long, but feel free to check out the [github](https://github.com/dghosef/FPL-team-generator/blob/main/src/pick_team.py) and leave any
feedback/questions.

