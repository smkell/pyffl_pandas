
# Introduction

At the onset of the 2016 fantasy football season I decided that I was going to try and win my league with numbers and computer science. I was going to take the opportunity to learn modern data science tools and finally put some of that college statistics I'd learned to use. 

A combination of factors led me to move through various tools and approaches at the beginning of the season. Ultimately I ended up using Excel to quickly fire up a list of players in the order I wanted by hand. In the next sectoin I'm goign to reproduce this methdology using python, specifically the data science package `pandas`.

# Python Reproduction of Excel Methodology

## Imports

These are the packages used to reproduce the list I used for drafting my team for 2016.

* `pandas` Is a Python data analysis library and is available [here](http://pandas.pydata.org/)
* `numpy` Is a package for scientific computing in python, used below primarily for its mathmatical functions and constructs. It is available [here](http://www.numpy.org/).
* `matplotlib.pyplot` Is a package for plotting data and is available [here](http://matplotlib.org/).
* `pyffl` Is a package I've developed for scraping fantasy football data and will be including any pure python functions there. It is available [here](https://github.com/smkell/pyffl)


```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from IPython.core.display import display

import pyffl
```

## Getting the data

My original intent was to use a wide variety of projection and ranking data to formulate my ranks. However, time, laziness and indecisiveness ulitmately led me to only using ESPN's standard projections. 

I've created a function in the `pyffl` package for retrieving projections from a variety of sources. Of course once again only `ESPN` is currently implemented.


```python
rules = pyffl.LeagueRules()
```


```python
projections = pyffl.scrape_projections(['espn'], 2016)
```

    
    Scraping ESPN projections for week 0 of 2016 season
    ...Done
    


```python
projections = pyffl.calculate_points(projections, rules)
```

## Building the dataframe

Pandas primarily operates on objects known as `Series` and `DataFrame` where a `DataFrame` is a table composed of several `Series` associated together by an `index`. In the below code segment we construct a `DataFram` for our projections. The `*_key` lists give the names of the columns in the desired order for display.


```python
df = pd.DataFrame(projections)
df['source'] = 'espn'
df['projection'] = True
df['vor'] = np.nan
df['rank'] = df['pts'].rank(ascending=0)
df['positionRank'] = 0.0

info_keys = ['rank', 'positionRank', 'name', 'team', 'position', 'source', 'projection']
skill_keys = ['passCmp', 'passAtt', 'passYds', 'passTds', 'passInts',
              'rushAtt', 'rushYds', 'rushTds',
              'recsCmp', 'recsAtt', 'recsYds', 'recsTds']
dst_keys = ['dstTckls', 'dstSacks', 'dstFmblFrc', 'dstFmblRec', 'dstInts', 'dstIntTds', 'dstFmblTds']
k_keys = ['fg0139Cmp', 'fg0139Att', 'fg4049Cmp', 'fg4049Att', 'fg50Cmp', 'fg50Att',
          'fgCmp', 'fgAtt', 'xpCmp', 'xpAtt']
calc_keys = ['pts','vor']
all_keys = info_keys + skill_keys + dst_keys + calc_keys
df = df[all_keys]
```

## Calculating VOR 

`Value over replacement` is a measure of a player's value compared to other players in that position. The theory here is that we can measure how valuable a player by comparing how many more points he is projected to score than the next startable player in the position. Analysing this number shows descrete gaps in value where players are split in tiers. We can, in principle, use this information to pick the most valuable players at the right moment in the draft.


```python
# Positions is a `dict` where the key is the position, and the value is the number of players in that position which could
# be started in any given week. I.e. if there are 8 teams in the league and one QB slot per team then 8 QBs could be started.
# Likewise if there are 2 RB slots and 1 RB/WR/TE Flex then at most 3*8(24) RBs could be started in a given week.
positions = {
    'QB': (rules.starting_qbs + rules.starting_superflex) * rules.num_teams,
    'RB': (rules.starting_rbs + rules.starting_superflex + rules.starting_flex) * rules.num_teams,
    'WR': (rules.starting_wrs + rules.starting_superflex + rules.starting_flex) * rules.num_teams,
    'TE': (rules.starting_tes + rules.starting_superflex + rules.starting_flex) * rules.num_teams,
    'K': rules.starting_ks * rules.num_teams,
    'D/ST': rules.starting_dst * rules.num_teams
}
```


```python
# Iterate through the positions, and calculate the vor for each player in the position.
for position, draftable in positions.iteritems():
    last_qb = df[df['position'] == position].sort_values(by='pts',ascending=False).iloc[draftable,:]
    df.ix[df['position'] == position, 'vor'] = df.ix[df['position'] == position, 'pts'] - last_qb['pts']
    df.ix[df['position'] == position, 'positionRank'] = df.ix[df['position'] == position, 'pts'].rank(ascending=0)
```

The following table are the top 16 players in all positions. It represents my rankings for the first two rounds of the 2016 draft.


```python
round = 0
start = round * rules.num_teams
end = start + rules.num_teams
df[['rank', 'positionRank', 'name', 'position', 'pts', 'vor']].sort_values(by='vor',ascending=False).iloc[start:end,:]
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>rank</th>
      <th>positionRank</th>
      <th>name</th>
      <th>position</th>
      <th>pts</th>
      <th>vor</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>320</th>
      <td>1.0</td>
      <td>1.0</td>
      <td>Antonio Brown</td>
      <td>WR</td>
      <td>383.00</td>
      <td>185.42</td>
    </tr>
    <tr>
      <th>322</th>
      <td>2.0</td>
      <td>2.0</td>
      <td>Julio Jones</td>
      <td>WR</td>
      <td>346.15</td>
      <td>148.57</td>
    </tr>
    <tr>
      <th>520</th>
      <td>47.0</td>
      <td>1.0</td>
      <td>Rob Gronkowski</td>
      <td>TE</td>
      <td>244.19</td>
      <td>140.42</td>
    </tr>
    <tr>
      <th>521</th>
      <td>55.0</td>
      <td>2.0</td>
      <td>Jordan Reed</td>
      <td>TE</td>
      <td>234.27</td>
      <td>130.50</td>
    </tr>
    <tr>
      <th>123</th>
      <td>13.0</td>
      <td>1.0</td>
      <td>David Johnson</td>
      <td>RB</td>
      <td>280.66</td>
      <td>120.97</td>
    </tr>
    <tr>
      <th>124</th>
      <td>19.0</td>
      <td>2.0</td>
      <td>Devonta Freeman</td>
      <td>RB</td>
      <td>270.52</td>
      <td>110.83</td>
    </tr>
    <tr>
      <th>121</th>
      <td>22.0</td>
      <td>3.0</td>
      <td>Todd Gurley</td>
      <td>RB</td>
      <td>269.01</td>
      <td>109.32</td>
    </tr>
    <tr>
      <th>321</th>
      <td>6.0</td>
      <td>3.0</td>
      <td>Odell Beckham Jr.</td>
      <td>WR</td>
      <td>305.81</td>
      <td>108.23</td>
    </tr>
  </tbody>
</table>
</div>




```python
round = 1
start = 1 + (round * rules.num_teams)
end = start + rules.num_teams
df[['rank', 'name', 'position', 'pts', 'vor']].sort_values(by='vor',ascending=False).iloc[start:end,:]
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>rank</th>
      <th>name</th>
      <th>position</th>
      <th>pts</th>
      <th>vor</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>120</th>
      <td>26.0</td>
      <td>Adrian Peterson</td>
      <td>RB</td>
      <td>264.50</td>
      <td>104.81</td>
    </tr>
    <tr>
      <th>524</th>
      <td>79.0</td>
      <td>Travis Kelce</td>
      <td>TE</td>
      <td>198.89</td>
      <td>95.12</td>
    </tr>
    <tr>
      <th>130</th>
      <td>36.0</td>
      <td>LeSean McCoy</td>
      <td>RB</td>
      <td>253.65</td>
      <td>93.96</td>
    </tr>
    <tr>
      <th>125</th>
      <td>37.0</td>
      <td>Le'Veon Bell</td>
      <td>RB</td>
      <td>252.79</td>
      <td>93.10</td>
    </tr>
    <tr>
      <th>523</th>
      <td>85.0</td>
      <td>Delanie Walker</td>
      <td>TE</td>
      <td>196.52</td>
      <td>92.75</td>
    </tr>
    <tr>
      <th>126</th>
      <td>39.0</td>
      <td>Lamar Miller</td>
      <td>RB</td>
      <td>251.80</td>
      <td>92.11</td>
    </tr>
    <tr>
      <th>122</th>
      <td>40.0</td>
      <td>Ezekiel Elliott</td>
      <td>RB</td>
      <td>251.62</td>
      <td>91.93</td>
    </tr>
    <tr>
      <th>323</th>
      <td>10.0</td>
      <td>DeAndre Hopkins</td>
      <td>WR</td>
      <td>284.82</td>
      <td>87.24</td>
    </tr>
  </tbody>
</table>
</div>



# A quick interlude for analysis

So far we've actually come pretty far in terms of data collection and massaging. I think, then that it's time that we start taking a look at what it all means. 

In the above table I've listed the 16 most valuable players according to my model. There are problems here. First of all going into the draft I never had any intention of drafting either Rob Gronkowski or Jordan Reed, simply based off the fact that I *knew* that the tight-end position simply wasn't valuable enough to justify a first or second round pick. The fact that two tightends show up in the top 16 suggest that either my assumptions, or the model, are wrong.

## Comparing projected rankings to actual rankings


```python
actuals = pyffl.scrape_actuals(2016)
actuals = pyffl.calculate_points(actuals, rules)
```


```python
dfa = pd.DataFrame(actuals, columns=all_keys)
dfa['vor'] = np.nan
dfa['rank'] = dfa['pts'].rank(ascending=0)
```


```python
positions = {
    'QB': (rules.starting_qbs + rules.starting_superflex) * rules.num_teams,
    'RB': (rules.starting_rbs + rules.starting_superflex + rules.starting_flex) * rules.num_teams,
    'WR': (rules.starting_wrs + rules.starting_superflex + rules.starting_flex) * rules.num_teams,
    'TE': (rules.starting_tes + rules.starting_superflex + rules.starting_flex) * rules.num_teams,
}
# Iterate through the positions, and calculate the vor for each player in the position.
for position, draftable in positions.iteritems():
    last_qb = dfa[dfa['position'] == position].sort_values(by='pts',ascending=False).iloc[draftable,:]
    dfa.ix[dfa['position'] == position, 'vor'] = dfa.ix[dfa['position'] == position, 'pts'] - last_qb['pts']
    dfa.ix[dfa['position'] == position, 'positionRank'] = dfa.ix[dfa['position'] == position, 'pts'].rank(ascending=0)
```

### Round 1 Projected vs Actual


```python
round = 0
start = round * rules.num_teams
end = start + rules.num_teams
display(df[['rank', 'positionRank', 'name', 'position', 'pts', 'vor']].sort_values(by='vor', ascending=False).iloc[start:end,:])
display(dfa[['rank', 'positionRank', 'name', 'position', 'pts', 'vor']].sort_values(by='vor',ascending=False).iloc[start:end,:])
```


<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>rank</th>
      <th>positionRank</th>
      <th>name</th>
      <th>position</th>
      <th>pts</th>
      <th>vor</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>320</th>
      <td>1.0</td>
      <td>1.0</td>
      <td>Antonio Brown</td>
      <td>WR</td>
      <td>383.00</td>
      <td>185.42</td>
    </tr>
    <tr>
      <th>322</th>
      <td>2.0</td>
      <td>2.0</td>
      <td>Julio Jones</td>
      <td>WR</td>
      <td>346.15</td>
      <td>148.57</td>
    </tr>
    <tr>
      <th>520</th>
      <td>47.0</td>
      <td>1.0</td>
      <td>Rob Gronkowski</td>
      <td>TE</td>
      <td>244.19</td>
      <td>140.42</td>
    </tr>
    <tr>
      <th>521</th>
      <td>55.0</td>
      <td>2.0</td>
      <td>Jordan Reed</td>
      <td>TE</td>
      <td>234.27</td>
      <td>130.50</td>
    </tr>
    <tr>
      <th>123</th>
      <td>13.0</td>
      <td>1.0</td>
      <td>David Johnson</td>
      <td>RB</td>
      <td>280.66</td>
      <td>120.97</td>
    </tr>
    <tr>
      <th>124</th>
      <td>19.0</td>
      <td>2.0</td>
      <td>Devonta Freeman</td>
      <td>RB</td>
      <td>270.52</td>
      <td>110.83</td>
    </tr>
    <tr>
      <th>121</th>
      <td>22.0</td>
      <td>3.0</td>
      <td>Todd Gurley</td>
      <td>RB</td>
      <td>269.01</td>
      <td>109.32</td>
    </tr>
    <tr>
      <th>321</th>
      <td>6.0</td>
      <td>3.0</td>
      <td>Odell Beckham Jr.</td>
      <td>WR</td>
      <td>305.81</td>
      <td>108.23</td>
    </tr>
  </tbody>
</table>
</div>



<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>rank</th>
      <th>positionRank</th>
      <th>name</th>
      <th>position</th>
      <th>pts</th>
      <th>vor</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>267</th>
      <td>1.0</td>
      <td>1.0</td>
      <td>David Johnson</td>
      <td>RB</td>
      <td>286.4</td>
      <td>185.6</td>
    </tr>
    <tr>
      <th>223</th>
      <td>2.0</td>
      <td>2.0</td>
      <td>Ezekiel Elliott</td>
      <td>RB</td>
      <td>266.7</td>
      <td>165.9</td>
    </tr>
    <tr>
      <th>183</th>
      <td>5.0</td>
      <td>3.0</td>
      <td>DeMarco Murray</td>
      <td>RB</td>
      <td>250.6</td>
      <td>149.8</td>
    </tr>
    <tr>
      <th>114</th>
      <td>11.0</td>
      <td>4.0</td>
      <td>Melvin Gordon</td>
      <td>RB</td>
      <td>230.3</td>
      <td>129.5</td>
    </tr>
    <tr>
      <th>302</th>
      <td>7.0</td>
      <td>1.0</td>
      <td>Antonio Brown</td>
      <td>WR</td>
      <td>242.7</td>
      <td>106.3</td>
    </tr>
    <tr>
      <th>33</th>
      <td>10.0</td>
      <td>2.0</td>
      <td>Mike Evans</td>
      <td>WR</td>
      <td>235.0</td>
      <td>98.6</td>
    </tr>
    <tr>
      <th>422</th>
      <td>22.0</td>
      <td>5.0</td>
      <td>Le'Veon Bell</td>
      <td>RB</td>
      <td>194.6</td>
      <td>93.8</td>
    </tr>
    <tr>
      <th>53</th>
      <td>28.0</td>
      <td>6.0</td>
      <td>LeSean McCoy</td>
      <td>RB</td>
      <td>188.4</td>
      <td>87.6</td>
    </tr>
  </tbody>
</table>
</div>


### Round 2 Projected vs. Actual


```python
round = 1
start = 1 + (round * rules.num_teams)
end = start + rules.num_teams
display(df[['rank', 'positionRank', 'name', 'position', 'pts', 'vor']].sort_values(by='vor', ascending=False).iloc[start:end,:])
display(dfa[['rank', 'positionRank', 'name', 'position', 'pts', 'vor']].sort_values(by='vor',ascending=False).iloc[start:end,:])
```


<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>rank</th>
      <th>positionRank</th>
      <th>name</th>
      <th>position</th>
      <th>pts</th>
      <th>vor</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>120</th>
      <td>26.0</td>
      <td>4.0</td>
      <td>Adrian Peterson</td>
      <td>RB</td>
      <td>264.50</td>
      <td>104.81</td>
    </tr>
    <tr>
      <th>524</th>
      <td>79.0</td>
      <td>4.0</td>
      <td>Travis Kelce</td>
      <td>TE</td>
      <td>198.89</td>
      <td>95.12</td>
    </tr>
    <tr>
      <th>130</th>
      <td>36.0</td>
      <td>5.0</td>
      <td>LeSean McCoy</td>
      <td>RB</td>
      <td>253.65</td>
      <td>93.96</td>
    </tr>
    <tr>
      <th>125</th>
      <td>37.0</td>
      <td>6.0</td>
      <td>Le'Veon Bell</td>
      <td>RB</td>
      <td>252.79</td>
      <td>93.10</td>
    </tr>
    <tr>
      <th>523</th>
      <td>85.0</td>
      <td>5.0</td>
      <td>Delanie Walker</td>
      <td>TE</td>
      <td>196.52</td>
      <td>92.75</td>
    </tr>
    <tr>
      <th>126</th>
      <td>39.0</td>
      <td>7.0</td>
      <td>Lamar Miller</td>
      <td>RB</td>
      <td>251.80</td>
      <td>92.11</td>
    </tr>
    <tr>
      <th>122</th>
      <td>40.0</td>
      <td>8.0</td>
      <td>Ezekiel Elliott</td>
      <td>RB</td>
      <td>251.62</td>
      <td>91.93</td>
    </tr>
    <tr>
      <th>323</th>
      <td>10.0</td>
      <td>4.0</td>
      <td>DeAndre Hopkins</td>
      <td>WR</td>
      <td>284.82</td>
      <td>87.24</td>
    </tr>
  </tbody>
</table>
</div>



<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>rank</th>
      <th>positionRank</th>
      <th>name</th>
      <th>position</th>
      <th>pts</th>
      <th>vor</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>92</th>
      <td>3.0</td>
      <td>1.0</td>
      <td>Aaron Rodgers</td>
      <td>QB</td>
      <td>263.46</td>
      <td>81.56</td>
    </tr>
    <tr>
      <th>293</th>
      <td>61.0</td>
      <td>2.0</td>
      <td>Jordan Reed</td>
      <td>TE</td>
      <td>152.00</td>
      <td>81.40</td>
    </tr>
    <tr>
      <th>184</th>
      <td>66.0</td>
      <td>3.0</td>
      <td>Delanie Walker</td>
      <td>TE</td>
      <td>151.00</td>
      <td>80.40</td>
    </tr>
    <tr>
      <th>123</th>
      <td>4.0</td>
      <td>2.0</td>
      <td>Drew Brees</td>
      <td>QB</td>
      <td>261.34</td>
      <td>79.44</td>
    </tr>
    <tr>
      <th>210</th>
      <td>73.0</td>
      <td>4.0</td>
      <td>Jimmy Graham</td>
      <td>TE</td>
      <td>145.60</td>
      <td>75.00</td>
    </tr>
    <tr>
      <th>19</th>
      <td>37.0</td>
      <td>7.0</td>
      <td>Devonta Freeman</td>
      <td>RB</td>
      <td>174.80</td>
      <td>74.00</td>
    </tr>
    <tr>
      <th>24</th>
      <td>13.0</td>
      <td>3.0</td>
      <td>Julio Jones</td>
      <td>WR</td>
      <td>209.00</td>
      <td>72.60</td>
    </tr>
    <tr>
      <th>109</th>
      <td>77.0</td>
      <td>5.0</td>
      <td>Travis Kelce</td>
      <td>TE</td>
      <td>142.00</td>
      <td>71.40</td>
    </tr>
  </tbody>
</table>
</div>


### Average projected points per position vs Average actual points per position


```python
num_rostered = (rules.starting_qbs + 
                rules.starting_rbs + 
                rules.starting_wrs + 
                rules.starting_tes + 
                rules.starting_ks + 
                rules.starting_dst + 
                rules.starting_flex +
                rules.starting_superflex +
                8)
grp_proj = df.sort_values(by='pts', ascending=False).iloc[0:num_rostered*8].groupby('position')
display(grp_proj.agg({'pts': np.mean}))

grp_actl = dfa.sort_values(by='pts', ascending=False).iloc[0:num_rostered*8].groupby('position')
display(grp_actl.agg({'pts': np.mean}))
```


<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>pts</th>
    </tr>
    <tr>
      <th>position</th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>QB</th>
      <td>262.582333</td>
    </tr>
    <tr>
      <th>RB</th>
      <td>201.517368</td>
    </tr>
    <tr>
      <th>TE</th>
      <td>180.324375</td>
    </tr>
    <tr>
      <th>WR</th>
      <td>207.957333</td>
    </tr>
  </tbody>
</table>
</div>



<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>pts</th>
    </tr>
    <tr>
      <th>position</th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>QB</th>
      <td>184.534667</td>
    </tr>
    <tr>
      <th>RB</th>
      <td>149.808571</td>
    </tr>
    <tr>
      <th>TE</th>
      <td>118.972222</td>
    </tr>
    <tr>
      <th>WR</th>
      <td>143.944918</td>
    </tr>
  </tbody>
</table>
</div>


### Identifying the biggest busts of the year.

The below table merges the actual and projected points, and calculates the difference between the two. The table is then sorted by the difference to identify the biggest "busts" or disappointments. Several of these we can attribute to injuries and we can discount them fairly easily. Others are less easy to explain.


```python
def color_negative_red(val):
    color = 'red' if val < 0 else 'black'
    return 'color: %s' % color

mrg = df.merge(dfa, left_on=['name','position'], right_on=['name','position'], how='inner', suffixes=['_proj', '_actl'])
mrg['pts_diff'] = mrg['pts_actl'] - mrg['pts_proj']
mrg['positionRank_diff'] = mrg['positionRank_proj'] - mrg['positionRank_actl']
mrg_cols = ['name', 'position','positionRank_proj', 'positionRank_actl', 'pts_proj', 'pts_actl', 'pts_diff', 'positionRank_diff']
tbl = mrg[mrg_cols].sort_values(by='pts_diff').iloc[0:rules.num_teams*3,:]
display(tbl.style.applymap(color_negative_red))
```



        <style  type="text/css" >
        
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row0_col0 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row0_col1 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row0_col2 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row0_col3 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row0_col4 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row0_col5 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row0_col6 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row0_col7 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row1_col0 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row1_col1 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row1_col2 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row1_col3 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row1_col4 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row1_col5 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row1_col6 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row1_col7 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row2_col0 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row2_col1 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row2_col2 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row2_col3 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row2_col4 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row2_col5 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row2_col6 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row2_col7 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row3_col0 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row3_col1 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row3_col2 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row3_col3 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row3_col4 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row3_col5 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row3_col6 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row3_col7 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row4_col0 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row4_col1 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row4_col2 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row4_col3 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row4_col4 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row4_col5 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row4_col6 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row4_col7 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row5_col0 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row5_col1 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row5_col2 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row5_col3 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row5_col4 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row5_col5 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row5_col6 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row5_col7 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row6_col0 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row6_col1 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row6_col2 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row6_col3 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row6_col4 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row6_col5 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row6_col6 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row6_col7 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row7_col0 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row7_col1 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row7_col2 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row7_col3 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row7_col4 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row7_col5 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row7_col6 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row7_col7 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row8_col0 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row8_col1 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row8_col2 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row8_col3 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row8_col4 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row8_col5 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row8_col6 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row8_col7 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row9_col0 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row9_col1 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row9_col2 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row9_col3 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row9_col4 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row9_col5 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row9_col6 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row9_col7 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row10_col0 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row10_col1 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row10_col2 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row10_col3 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row10_col4 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row10_col5 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row10_col6 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row10_col7 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row11_col0 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row11_col1 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row11_col2 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row11_col3 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row11_col4 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row11_col5 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row11_col6 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row11_col7 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row12_col0 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row12_col1 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row12_col2 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row12_col3 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row12_col4 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row12_col5 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row12_col6 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row12_col7 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row13_col0 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row13_col1 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row13_col2 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row13_col3 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row13_col4 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row13_col5 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row13_col6 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row13_col7 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row14_col0 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row14_col1 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row14_col2 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row14_col3 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row14_col4 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row14_col5 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row14_col6 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row14_col7 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row15_col0 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row15_col1 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row15_col2 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row15_col3 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row15_col4 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row15_col5 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row15_col6 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row15_col7 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row16_col0 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row16_col1 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row16_col2 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row16_col3 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row16_col4 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row16_col5 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row16_col6 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row16_col7 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row17_col0 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row17_col1 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row17_col2 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row17_col3 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row17_col4 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row17_col5 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row17_col6 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row17_col7 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row18_col0 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row18_col1 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row18_col2 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row18_col3 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row18_col4 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row18_col5 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row18_col6 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row18_col7 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row19_col0 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row19_col1 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row19_col2 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row19_col3 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row19_col4 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row19_col5 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row19_col6 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row19_col7 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row20_col0 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row20_col1 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row20_col2 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row20_col3 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row20_col4 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row20_col5 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row20_col6 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row20_col7 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row21_col0 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row21_col1 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row21_col2 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row21_col3 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row21_col4 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row21_col5 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row21_col6 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row21_col7 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row22_col0 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row22_col1 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row22_col2 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row22_col3 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row22_col4 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row22_col5 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row22_col6 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row22_col7 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row23_col0 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row23_col1 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row23_col2 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row23_col3 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row23_col4 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row23_col5 {
            
                color:  black;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row23_col6 {
            
                color:  red;
            
            }
        
            #T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row23_col7 {
            
                color:  red;
            
            }
        
        </style>

        <table id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4" None>
        

        <thead>
            
            <tr>
                
                
                <th class="blank level0" >
                  
                
                
                
                <th class="col_heading level0 col0" colspan=1>
                  name
                
                
                
                <th class="col_heading level0 col1" colspan=1>
                  position
                
                
                
                <th class="col_heading level0 col2" colspan=1>
                  positionRank_proj
                
                
                
                <th class="col_heading level0 col3" colspan=1>
                  positionRank_actl
                
                
                
                <th class="col_heading level0 col4" colspan=1>
                  pts_proj
                
                
                
                <th class="col_heading level0 col5" colspan=1>
                  pts_actl
                
                
                
                <th class="col_heading level0 col6" colspan=1>
                  pts_diff
                
                
                
                <th class="col_heading level0 col7" colspan=1>
                  positionRank_diff
                
                
            </tr>
            
        </thead>
        <tbody>
            
            <tr>
                
                
                <th id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4"
                 class="row_heading level0 row0" rowspan=1>
                    60
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row0_col0"
                 class="data row0 col0" >
                    Adrian Peterson
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row0_col1"
                 class="data row0 col1" >
                    RB
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row0_col2"
                 class="data row0 col2" >
                    4
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row0_col3"
                 class="data row0 col3" >
                    102
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row0_col4"
                 class="data row0 col4" >
                    264.5
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row0_col5"
                 class="data row0 col5" >
                    7.7
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row0_col6"
                 class="data row0 col6" >
                    -256.8
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row0_col7"
                 class="data row0 col7" >
                    -98
                
                
            </tr>
            
            <tr>
                
                
                <th id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4"
                 class="row_heading level0 row1" rowspan=1>
                    185
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row1_col0"
                 class="data row1 col0" >
                    Keenan Allen
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row1_col1"
                 class="data row1 col1" >
                    WR
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row1_col2"
                 class="data row1 col2" >
                    8
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row1_col3"
                 class="data row1 col3" >
                    142
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row1_col4"
                 class="data row1 col4" >
                    264.36
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row1_col5"
                 class="data row1 col5" >
                    12.3
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row1_col6"
                 class="data row1 col6" >
                    -252.06
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row1_col7"
                 class="data row1 col7" >
                    -134
                
                
            </tr>
            
            <tr>
                
                
                <th id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4"
                 class="row_heading level0 row2" rowspan=1>
                    24
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row2_col0"
                 class="data row2 col0" >
                    Robert Griffin III
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row2_col1"
                 class="data row2 col1" >
                    QB
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row2_col2"
                 class="data row2 col2" >
                    20
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row2_col3"
                 class="data row2 col3" >
                    45
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row2_col4"
                 class="data row2 col4" >
                    253.734
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row2_col5"
                 class="data row2 col5" >
                    9.3
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row2_col6"
                 class="data row2 col6" >
                    -244.434
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row2_col7"
                 class="data row2 col7" >
                    -25
                
                
            </tr>
            
            <tr>
                
                
                <th id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4"
                 class="row_heading level0 row3" rowspan=1>
                    181
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row3_col0"
                 class="data row3 col0" >
                    Sammy Watkins
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row3_col1"
                 class="data row3 col1" >
                    WR
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row3_col2"
                 class="data row3 col2" >
                    17
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row3_col3"
                 class="data row3 col3" >
                    129
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row3_col4"
                 class="data row3 col4" >
                    237.17
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row3_col5"
                 class="data row3 col5" >
                    23.3
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row3_col6"
                 class="data row3 col6" >
                    -213.87
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row3_col7"
                 class="data row3 col7" >
                    -112
                
                
            </tr>
            
            <tr>
                
                
                <th id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4"
                 class="row_heading level0 row4" rowspan=1>
                    192
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row4_col0"
                 class="data row4 col0" >
                    Eric Decker
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row4_col1"
                 class="data row4 col1" >
                    WR
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row4_col2"
                 class="data row4 col2" >
                    12
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row4_col3"
                 class="data row4 col3" >
                    103
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row4_col4"
                 class="data row4 col4" >
                    250.94
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row4_col5"
                 class="data row4 col5" >
                    40.4
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row4_col6"
                 class="data row4 col6" >
                    -210.54
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row4_col7"
                 class="data row4 col7" >
                    -91
                
                
            </tr>
            
            <tr>
                
                
                <th id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4"
                 class="row_heading level0 row5" rowspan=1>
                    74
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row5_col0"
                 class="data row5 col0" >
                    Jamaal Charles
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row5_col1"
                 class="data row5 col1" >
                    RB
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row5_col2"
                 class="data row5 col2" >
                    12
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row5_col3"
                 class="data row5 col3" >
                    94
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row5_col4"
                 class="data row5 col4" >
                    221.65
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row5_col5"
                 class="data row5 col5" >
                    13.4
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row5_col6"
                 class="data row5 col6" >
                    -208.25
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row5_col7"
                 class="data row5 col7" >
                    -82
                
                
            </tr>
            
            <tr>
                
                
                <th id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4"
                 class="row_heading level0 row6" rowspan=1>
                    84
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row6_col0"
                 class="data row6 col0" >
                    Danny Woodhead
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row6_col1"
                 class="data row6 col1" >
                    RB
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row6_col2"
                 class="data row6 col2" >
                    13
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row6_col3"
                 class="data row6 col3" >
                    75
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row6_col4"
                 class="data row6 col4" >
                    211.5
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row6_col5"
                 class="data row6 col5" >
                    27.1
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row6_col6"
                 class="data row6 col6" >
                    -184.4
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row6_col7"
                 class="data row6 col7" >
                    -62
                
                
            </tr>
            
            <tr>
                
                
                <th id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4"
                 class="row_heading level0 row7" rowspan=1>
                    23
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row7_col0"
                 class="data row7 col0" >
                    Jay Cutler
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row7_col1"
                 class="data row7 col1" >
                    QB
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row7_col2"
                 class="data row7 col2" >
                    26
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row7_col3"
                 class="data row7 col3" >
                    35
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row7_col4"
                 class="data row7 col4" >
                    229.376
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row7_col5"
                 class="data row7 col5" >
                    50.76
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row7_col6"
                 class="data row7 col6" >
                    -178.616
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row7_col7"
                 class="data row7 col7" >
                    -9
                
                
            </tr>
            
            <tr>
                
                
                <th id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4"
                 class="row_heading level0 row8" rowspan=1>
                    67
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row8_col0"
                 class="data row8 col0" >
                    Doug Martin
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row8_col1"
                 class="data row8 col1" >
                    RB
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row8_col2"
                 class="data row8 col2" >
                    10
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row8_col3"
                 class="data row8 col3" >
                    54
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row8_col4"
                 class="data row8 col4" >
                    224.65
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row8_col5"
                 class="data row8 col5" >
                    53
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row8_col6"
                 class="data row8 col6" >
                    -171.65
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row8_col7"
                 class="data row8 col7" >
                    -44
                
                
            </tr>
            
            <tr>
                
                
                <th id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4"
                 class="row_heading level0 row9" rowspan=1>
                    90
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row9_col0"
                 class="data row9 col0" >
                    Ameer Abdullah
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row9_col1"
                 class="data row9 col1" >
                    RB
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row9_col2"
                 class="data row9 col2" >
                    20
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row9_col3"
                 class="data row9 col3" >
                    76
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row9_col4"
                 class="data row9 col4" >
                    190.47
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row9_col5"
                 class="data row9 col5" >
                    26.8
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row9_col6"
                 class="data row9 col6" >
                    -163.67
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row9_col7"
                 class="data row9 col7" >
                    -56
                
                
            </tr>
            
            <tr>
                
                
                <th id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4"
                 class="row_heading level0 row10" rowspan=1>
                    69
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row10_col0"
                 class="data row10 col0" >
                    Eddie Lacy
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row10_col1"
                 class="data row10 col1" >
                    RB
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row10_col2"
                 class="data row10 col2" >
                    17
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row10_col3"
                 class="data row10 col3" >
                    61
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row10_col4"
                 class="data row10 col4" >
                    200.01
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row10_col5"
                 class="data row10 col5" >
                    42.8
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row10_col6"
                 class="data row10 col6" >
                    -157.21
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row10_col7"
                 class="data row10 col7" >
                    -44
                
                
            </tr>
            
            <tr>
                
                
                <th id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4"
                 class="row_heading level0 row11" rowspan=1>
                    190
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row11_col0"
                 class="data row11 col0" >
                    Jeremy Maclin
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row11_col1"
                 class="data row11 col1" >
                    WR
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row11_col2"
                 class="data row11 col2" >
                    18
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row11_col3"
                 class="data row11 col3" >
                    72
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row11_col4"
                 class="data row11 col4" >
                    235.99
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row11_col5"
                 class="data row11 col5" >
                    79.5
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row11_col6"
                 class="data row11 col6" >
                    -156.49
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row11_col7"
                 class="data row11 col7" >
                    -54
                
                
            </tr>
            
            <tr>
                
                
                <th id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4"
                 class="row_heading level0 row12" rowspan=1>
                    179
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row12_col0"
                 class="data row12 col0" >
                    Alshon Jeffery
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row12_col1"
                 class="data row12 col1" >
                    WR
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row12_col2"
                 class="data row12 col2" >
                    9
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row12_col3"
                 class="data row12 col3" >
                    49
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row12_col4"
                 class="data row12 col4" >
                    261.53
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row12_col5"
                 class="data row12 col5" >
                    109
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row12_col6"
                 class="data row12 col6" >
                    -152.53
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row12_col7"
                 class="data row12 col7" >
                    -40
                
                
            </tr>
            
            <tr>
                
                
                <th id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4"
                 class="row_heading level0 row13" rowspan=1>
                    173
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row13_col0"
                 class="data row13 col0" >
                    DeAndre Hopkins
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row13_col1"
                 class="data row13 col1" >
                    WR
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row13_col2"
                 class="data row13 col2" >
                    4
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row13_col3"
                 class="data row13 col3" >
                    35
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row13_col4"
                 class="data row13 col4" >
                    284.82
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row13_col5"
                 class="data row13 col5" >
                    134
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row13_col6"
                 class="data row13 col6" >
                    -150.82
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row13_col7"
                 class="data row13 col7" >
                    -31
                
                
            </tr>
            
            <tr>
                
                
                <th id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4"
                 class="row_heading level0 row14" rowspan=1>
                    18
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row14_col0"
                 class="data row14 col0" >
                    Ryan Fitzpatrick
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row14_col1"
                 class="data row14 col1" >
                    QB
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row14_col2"
                 class="data row14 col2" >
                    15
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row14_col3"
                 class="data row14 col3" >
                    29
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row14_col4"
                 class="data row14 col4" >
                    265.806
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row14_col5"
                 class="data row14 col5" >
                    115.34
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row14_col6"
                 class="data row14 col6" >
                    -150.466
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row14_col7"
                 class="data row14 col7" >
                    -14
                
                
            </tr>
            
            <tr>
                
                
                <th id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4"
                 class="row_heading level0 row15" rowspan=1>
                    0
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row15_col0"
                 class="data row15 col0" >
                    Cam Newton
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row15_col1"
                 class="data row15 col1" >
                    QB
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row15_col2"
                 class="data row15 col2" >
                    1
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row15_col3"
                 class="data row15 col3" >
                    16
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row15_col4"
                 class="data row15 col4" >
                    340.74
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row15_col5"
                 class="data row15 col5" >
                    190.68
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row15_col6"
                 class="data row15 col6" >
                    -150.06
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row15_col7"
                 class="data row15 col7" >
                    -15
                
                
            </tr>
            
            <tr>
                
                
                <th id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4"
                 class="row_heading level0 row16" rowspan=1>
                    89
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row16_col0"
                 class="data row16 col0" >
                    Arian Foster
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row16_col1"
                 class="data row16 col1" >
                    RB
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row16_col2"
                 class="data row16 col2" >
                    31
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row16_col3"
                 class="data row16 col3" >
                    86
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row16_col4"
                 class="data row16 col4" >
                    168.54
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row16_col5"
                 class="data row16 col5" >
                    19.3
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row16_col6"
                 class="data row16 col6" >
                    -149.24
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row16_col7"
                 class="data row16 col7" >
                    -55
                
                
            </tr>
            
            <tr>
                
                
                <th id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4"
                 class="row_heading level0 row17" rowspan=1>
                    296
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row17_col0"
                 class="data row17 col0" >
                    Rob Gronkowski
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row17_col1"
                 class="data row17 col1" >
                    TE
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row17_col2"
                 class="data row17 col2" >
                    1
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row17_col3"
                 class="data row17 col3" >
                    14
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row17_col4"
                 class="data row17 col4" >
                    244.19
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row17_col5"
                 class="data row17 col5" >
                    97
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row17_col6"
                 class="data row17 col6" >
                    -147.19
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row17_col7"
                 class="data row17 col7" >
                    -13
                
                
            </tr>
            
            <tr>
                
                
                <th id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4"
                 class="row_heading level0 row18" rowspan=1>
                    81
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row18_col0"
                 class="data row18 col0" >
                    Jeremy Langford
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row18_col1"
                 class="data row18 col1" >
                    RB
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row18_col2"
                 class="data row18 col2" >
                    19
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row18_col3"
                 class="data row18 col3" >
                    53
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row18_col4"
                 class="data row18 col4" >
                    197.32
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row18_col5"
                 class="data row18 col5" >
                    53.3
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row18_col6"
                 class="data row18 col6" >
                    -144.02
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row18_col7"
                 class="data row18 col7" >
                    -34
                
                
            </tr>
            
            <tr>
                
                
                <th id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4"
                 class="row_heading level0 row19" rowspan=1>
                    194
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row19_col0"
                 class="data row19 col0" >
                    Donte Moncrief
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row19_col1"
                 class="data row19 col1" >
                    WR
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row19_col2"
                 class="data row19 col2" >
                    25
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row19_col3"
                 class="data row19 col3" >
                    74
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row19_col4"
                 class="data row19 col4" >
                    221.91
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row19_col5"
                 class="data row19 col5" >
                    79
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row19_col6"
                 class="data row19 col6" >
                    -142.91
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row19_col7"
                 class="data row19 col7" >
                    -49
                
                
            </tr>
            
            <tr>
                
                
                <th id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4"
                 class="row_heading level0 row20" rowspan=1>
                    178
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row20_col0"
                 class="data row20 col0" >
                    Brandon Marshall
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row20_col1"
                 class="data row20 col1" >
                    WR
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row20_col2"
                 class="data row20 col2" >
                    5
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row20_col3"
                 class="data row20 col3" >
                    34
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row20_col4"
                 class="data row20 col4" >
                    275.95
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row20_col5"
                 class="data row20 col5" >
                    134.7
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row20_col6"
                 class="data row20 col6" >
                    -141.25
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row20_col7"
                 class="data row20 col7" >
                    -29
                
                
            </tr>
            
            <tr>
                
                
                <th id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4"
                 class="row_heading level0 row21" rowspan=1>
                    171
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row21_col0"
                 class="data row21 col0" >
                    Antonio Brown
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row21_col1"
                 class="data row21 col1" >
                    WR
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row21_col2"
                 class="data row21 col2" >
                    1
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row21_col3"
                 class="data row21 col3" >
                    1
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row21_col4"
                 class="data row21 col4" >
                    383
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row21_col5"
                 class="data row21 col5" >
                    242.7
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row21_col6"
                 class="data row21 col6" >
                    -140.3
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row21_col7"
                 class="data row21 col7" >
                    0
                
                
            </tr>
            
            <tr>
                
                
                <th id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4"
                 class="row_heading level0 row22" rowspan=1>
                    172
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row22_col0"
                 class="data row22 col0" >
                    Julio Jones
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row22_col1"
                 class="data row22 col1" >
                    WR
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row22_col2"
                 class="data row22 col2" >
                    2
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row22_col3"
                 class="data row22 col3" >
                    3
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row22_col4"
                 class="data row22 col4" >
                    346.15
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row22_col5"
                 class="data row22 col5" >
                    209
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row22_col6"
                 class="data row22 col6" >
                    -137.15
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row22_col7"
                 class="data row22 col7" >
                    -1
                
                
            </tr>
            
            <tr>
                
                
                <th id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4"
                 class="row_heading level0 row23" rowspan=1>
                    75
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row23_col0"
                 class="data row23 col0" >
                    Thomas Rawls
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row23_col1"
                 class="data row23 col1" >
                    RB
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row23_col2"
                 class="data row23 col2" >
                    32
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row23_col3"
                 class="data row23 col3" >
                    73
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row23_col4"
                 class="data row23 col4" >
                    164.19
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row23_col5"
                 class="data row23 col5" >
                    28.2
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row23_col6"
                 class="data row23 col6" >
                    -135.99
                
                
                
                <td id="T_70b8a1cf_b8d2_11e6_82d7_708bcdbedcd4row23_col7"
                 class="data row23 col7" >
                    -41
                
                
            </tr>
            
        </tbody>
        </table>
        


## Point Dropoff By Position

This chart displays the number of points missed out on by drafting the last starter in the position, rather than the best. "The Fantasy Football Guys" call this the magic formula.


```python
grp_proj = dfa.sort_values(by='vor', ascending=False).groupby('position')
tbl = grp_proj.agg({'vor': np.max}).sort_values(by='vor', ascending=False)
tbl['vor_per_week'] = tbl['vor'] / 13
display(tbl)
```


<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>vor</th>
      <th>vor_per_week</th>
    </tr>
    <tr>
      <th>position</th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>RB</th>
      <td>185.60</td>
      <td>14.276923</td>
    </tr>
    <tr>
      <th>WR</th>
      <td>106.30</td>
      <td>8.176923</td>
    </tr>
    <tr>
      <th>TE</th>
      <td>84.40</td>
      <td>6.492308</td>
    </tr>
    <tr>
      <th>QB</th>
      <td>81.56</td>
      <td>6.273846</td>
    </tr>
    <tr>
      <th>K</th>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>


This fairly straightforward table provides a fair amount of valuable information.


```python

winning_scores = pd.DataFrame([
        { 'week': 1, 'home': 'KELL', 'away': 'PINK', 'home_score': 115.3, 'away_score': 111.7},
        { 'week': 1, 'home': 'ARTH', 'away': 'TAUR', 'home_score': 117.8, 'away_score': 117.7},
        { 'week': 1, 'home': 'OBRY', 'away': 'TUCK', 'home_score': 202.1, 'away_score': 138.6},
        { 'week': 1, 'home': 'HMBE', 'away': 'SING', 'home_score': 150.6, 'away_score': 110.5},
        { 'week': 2, 'home': 'ARTH', 'away': 'TUCK', 'home_score': 115.9, 'away_score': 97.3},
        { 'week': 2, 'home': 'PINK', 'away': 'TAUR', 'home_score': 100.3, 'away_score': 151.4},
        { 'week': 2, 'home': 'HMBE', 'away': 'OBRY', 'home_score': 117, 'away_score': 100.3},
        { 'week': 2, 'home': 'TUCK', 'away': 'SING', 'home_score': 136.4, 'away_score': 90.4},
        { 'week': 3, 'home': 'KELL', 'away': 'TAUR', 'home_score': 114.3, 'away_score': 167},
        { 'week': 3, 'home': 'PINK', 'away': 'ARTH', 'home_score': 126.7, 'away_score': 119.6},
        { 'week': 3, 'home': 'OBRY', 'away': 'SING', 'home_score': 92.7, 'away_score': 159.4},
        { 'week': 3, 'home': 'TUCK', 'away': 'HMBE', 'home_score': 100.5, 'away_score': 140.8},
        { 'week': 4, 'home': 'OBRY', 'away': 'KELL', 'home_score': 110.5, 'away_score': 84.3},
        { 'week': 4, 'home': 'TUCK', 'away': 'PINK', 'home_score': 183.4, 'away_score': 124.3},
        { 'week': 4, 'home': 'HMBE', 'away': 'ARTH', 'home_score': 151, 'away_score': 96.1},
        { 'week': 4, 'home': 'SING', 'away': 'TAUR', 'home_score': 109.4, 'away_score': 76.7},
        { 'week': 4, 'home': 'SING', 'away': 'TAUR', 'home_score': 109.4, 'away_score': 76.7},
        { 'week': 5, 'home': 'KELL', 'away': 'SING', 'home_score': 122.1, 'away_score': 132.6},
        { 'week': 5, 'home': 'PINK', 'away': 'OBRY', 'home_score': 144.6, 'away_score': 88.7},
        { 'week': 5, 'home': 'ARTH', 'away': 'TUCK', 'home_score': 154.8, 'away_score': 120.6},
        { 'week': 5, 'home': 'TAUR', 'away': 'HMBE', 'home_score': 156.7, 'away_score': 163.3},
        { 'week': 6, 'home': 'HMBE', 'away': 'KELL', 'home_score': 89.7, 'away_score': 139.2},
        { 'week': 6, 'home': 'OBRY', 'away': 'ARTH', 'home_score': 132.8, 'away_score': 138.4},
        { 'week': 6, 'home': 'TUCK', 'away': 'TAUR', 'home_score': 156.4, 'away_score': 130.5},
        { 'week': 6, 'home': 'SING', 'away': 'PINK', 'home_score': 119.1, 'away_score': 110.4},
        { 'week': 7, 'home': 'KELL', 'away': 'TUCK', 'home_score': 114.5, 'away_score': 133.2},
        { 'week': 7, 'home': 'TAUR', 'away': 'OBRY', 'home_score': 97.5, 'away_score': 124.4},
        { 'week': 7, 'home': 'PINK', 'away': 'HMBE', 'home_score': 130.7, 'away_score': 133.3},
        { 'week': 7, 'home': 'ARTH', 'away': 'SING', 'home_score': 148.5, 'away_score': 95.4}
    ], columns=['week', 'home', 'away', 'home_score', 'away_score'])
winning_scores['winning_score'] = winning_scores.loc[:, ['home_score', 'away_score']].max(axis=1)
winning_scores['margin_of_victory'] = winning_scores['winning_score'] - winning_scores.loc[:, ['home_score', 'away_score']].min(axis=1)
display(winning_scores)
```


<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>week</th>
      <th>home</th>
      <th>away</th>
      <th>home_score</th>
      <th>away_score</th>
      <th>winning_score</th>
      <th>margin_of_victory</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>KELL</td>
      <td>PINK</td>
      <td>115.3</td>
      <td>111.7</td>
      <td>115.3</td>
      <td>3.6</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1</td>
      <td>ARTH</td>
      <td>TAUR</td>
      <td>117.8</td>
      <td>117.7</td>
      <td>117.8</td>
      <td>0.1</td>
    </tr>
    <tr>
      <th>2</th>
      <td>1</td>
      <td>OBRY</td>
      <td>TUCK</td>
      <td>202.1</td>
      <td>138.6</td>
      <td>202.1</td>
      <td>63.5</td>
    </tr>
    <tr>
      <th>3</th>
      <td>1</td>
      <td>HMBE</td>
      <td>SING</td>
      <td>150.6</td>
      <td>110.5</td>
      <td>150.6</td>
      <td>40.1</td>
    </tr>
    <tr>
      <th>4</th>
      <td>2</td>
      <td>ARTH</td>
      <td>TUCK</td>
      <td>115.9</td>
      <td>97.3</td>
      <td>115.9</td>
      <td>18.6</td>
    </tr>
    <tr>
      <th>5</th>
      <td>2</td>
      <td>PINK</td>
      <td>TAUR</td>
      <td>100.3</td>
      <td>151.4</td>
      <td>151.4</td>
      <td>51.1</td>
    </tr>
    <tr>
      <th>6</th>
      <td>2</td>
      <td>HMBE</td>
      <td>OBRY</td>
      <td>117.0</td>
      <td>100.3</td>
      <td>117.0</td>
      <td>16.7</td>
    </tr>
    <tr>
      <th>7</th>
      <td>2</td>
      <td>TUCK</td>
      <td>SING</td>
      <td>136.4</td>
      <td>90.4</td>
      <td>136.4</td>
      <td>46.0</td>
    </tr>
    <tr>
      <th>8</th>
      <td>3</td>
      <td>KELL</td>
      <td>TAUR</td>
      <td>114.3</td>
      <td>167.0</td>
      <td>167.0</td>
      <td>52.7</td>
    </tr>
    <tr>
      <th>9</th>
      <td>3</td>
      <td>PINK</td>
      <td>ARTH</td>
      <td>126.7</td>
      <td>119.6</td>
      <td>126.7</td>
      <td>7.1</td>
    </tr>
    <tr>
      <th>10</th>
      <td>3</td>
      <td>OBRY</td>
      <td>SING</td>
      <td>92.7</td>
      <td>159.4</td>
      <td>159.4</td>
      <td>66.7</td>
    </tr>
    <tr>
      <th>11</th>
      <td>3</td>
      <td>TUCK</td>
      <td>HMBE</td>
      <td>100.5</td>
      <td>140.8</td>
      <td>140.8</td>
      <td>40.3</td>
    </tr>
    <tr>
      <th>12</th>
      <td>4</td>
      <td>OBRY</td>
      <td>KELL</td>
      <td>110.5</td>
      <td>84.3</td>
      <td>110.5</td>
      <td>26.2</td>
    </tr>
    <tr>
      <th>13</th>
      <td>4</td>
      <td>TUCK</td>
      <td>PINK</td>
      <td>183.4</td>
      <td>124.3</td>
      <td>183.4</td>
      <td>59.1</td>
    </tr>
    <tr>
      <th>14</th>
      <td>4</td>
      <td>HMBE</td>
      <td>ARTH</td>
      <td>151.0</td>
      <td>96.1</td>
      <td>151.0</td>
      <td>54.9</td>
    </tr>
    <tr>
      <th>15</th>
      <td>4</td>
      <td>SING</td>
      <td>TAUR</td>
      <td>109.4</td>
      <td>76.7</td>
      <td>109.4</td>
      <td>32.7</td>
    </tr>
    <tr>
      <th>16</th>
      <td>4</td>
      <td>SING</td>
      <td>TAUR</td>
      <td>109.4</td>
      <td>76.7</td>
      <td>109.4</td>
      <td>32.7</td>
    </tr>
    <tr>
      <th>17</th>
      <td>5</td>
      <td>KELL</td>
      <td>SING</td>
      <td>122.1</td>
      <td>132.6</td>
      <td>132.6</td>
      <td>10.5</td>
    </tr>
    <tr>
      <th>18</th>
      <td>5</td>
      <td>PINK</td>
      <td>OBRY</td>
      <td>144.6</td>
      <td>88.7</td>
      <td>144.6</td>
      <td>55.9</td>
    </tr>
    <tr>
      <th>19</th>
      <td>5</td>
      <td>ARTH</td>
      <td>TUCK</td>
      <td>154.8</td>
      <td>120.6</td>
      <td>154.8</td>
      <td>34.2</td>
    </tr>
    <tr>
      <th>20</th>
      <td>5</td>
      <td>TAUR</td>
      <td>HMBE</td>
      <td>156.7</td>
      <td>163.3</td>
      <td>163.3</td>
      <td>6.6</td>
    </tr>
    <tr>
      <th>21</th>
      <td>6</td>
      <td>HMBE</td>
      <td>KELL</td>
      <td>89.7</td>
      <td>139.2</td>
      <td>139.2</td>
      <td>49.5</td>
    </tr>
    <tr>
      <th>22</th>
      <td>6</td>
      <td>OBRY</td>
      <td>ARTH</td>
      <td>132.8</td>
      <td>138.4</td>
      <td>138.4</td>
      <td>5.6</td>
    </tr>
    <tr>
      <th>23</th>
      <td>6</td>
      <td>TUCK</td>
      <td>TAUR</td>
      <td>156.4</td>
      <td>130.5</td>
      <td>156.4</td>
      <td>25.9</td>
    </tr>
    <tr>
      <th>24</th>
      <td>6</td>
      <td>SING</td>
      <td>PINK</td>
      <td>119.1</td>
      <td>110.4</td>
      <td>119.1</td>
      <td>8.7</td>
    </tr>
    <tr>
      <th>25</th>
      <td>7</td>
      <td>KELL</td>
      <td>TUCK</td>
      <td>114.5</td>
      <td>133.2</td>
      <td>133.2</td>
      <td>18.7</td>
    </tr>
    <tr>
      <th>26</th>
      <td>7</td>
      <td>TAUR</td>
      <td>OBRY</td>
      <td>97.5</td>
      <td>124.4</td>
      <td>124.4</td>
      <td>26.9</td>
    </tr>
    <tr>
      <th>27</th>
      <td>7</td>
      <td>PINK</td>
      <td>HMBE</td>
      <td>130.7</td>
      <td>133.3</td>
      <td>133.3</td>
      <td>2.6</td>
    </tr>
    <tr>
      <th>28</th>
      <td>7</td>
      <td>ARTH</td>
      <td>SING</td>
      <td>148.5</td>
      <td>95.4</td>
      <td>148.5</td>
      <td>53.1</td>
    </tr>
  </tbody>
</table>
</div>



```python
grp_proj = winning_scores.groupby('week')
col_names = {'winning_score': 'Winning Score', 
             'margin_of_victory': 'Margin of Victory'}
display(grp_proj.agg({'winning_score': [np.mean, np.min, np.max],
                      'margin_of_victory': [np.mean, np.min, np.max]
                     }).rename(columns=col_names))
```


<div>
<table border="1" class="dataframe">
  <thead>
    <tr>
      <th></th>
      <th colspan="3" halign="left">Winning Score</th>
      <th colspan="3" halign="left">Margin of Victory</th>
    </tr>
    <tr>
      <th></th>
      <th>mean</th>
      <th>amin</th>
      <th>amax</th>
      <th>mean</th>
      <th>amin</th>
      <th>amax</th>
    </tr>
    <tr>
      <th>week</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1</th>
      <td>146.450</td>
      <td>115.3</td>
      <td>202.1</td>
      <td>26.825</td>
      <td>0.1</td>
      <td>63.5</td>
    </tr>
    <tr>
      <th>2</th>
      <td>130.175</td>
      <td>115.9</td>
      <td>151.4</td>
      <td>33.100</td>
      <td>16.7</td>
      <td>51.1</td>
    </tr>
    <tr>
      <th>3</th>
      <td>148.475</td>
      <td>126.7</td>
      <td>167.0</td>
      <td>41.700</td>
      <td>7.1</td>
      <td>66.7</td>
    </tr>
    <tr>
      <th>4</th>
      <td>132.740</td>
      <td>109.4</td>
      <td>183.4</td>
      <td>41.120</td>
      <td>26.2</td>
      <td>59.1</td>
    </tr>
    <tr>
      <th>5</th>
      <td>148.825</td>
      <td>132.6</td>
      <td>163.3</td>
      <td>26.800</td>
      <td>6.6</td>
      <td>55.9</td>
    </tr>
    <tr>
      <th>6</th>
      <td>138.275</td>
      <td>119.1</td>
      <td>156.4</td>
      <td>22.425</td>
      <td>5.6</td>
      <td>49.5</td>
    </tr>
    <tr>
      <th>7</th>
      <td>134.850</td>
      <td>124.4</td>
      <td>148.5</td>
      <td>25.325</td>
      <td>2.6</td>
      <td>53.1</td>
    </tr>
  </tbody>
</table>
</div>



```python
mean_margin_of_victory = grp_proj['margin_of_victory'].mean().mean()
stdev_margin_of_victory = grp_proj['margin_of_victory'].mean().std()
print(mean_margin_of_victory, stdev_margin_of_victory)
tbl['pct_margin_victory'] = (tbl['vor_per_week'] / mean_margin_of_victory) * 100
display(tbl)
```

    (31.04214285714286, 7.7686211009112718)
    


<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>vor</th>
      <th>vor_per_week</th>
      <th>pct_margin_victory</th>
    </tr>
    <tr>
      <th>position</th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>RB</th>
      <td>185.60</td>
      <td>14.276923</td>
      <td>45.992067</td>
    </tr>
    <tr>
      <th>WR</th>
      <td>106.30</td>
      <td>8.176923</td>
      <td>26.341362</td>
    </tr>
    <tr>
      <th>TE</th>
      <td>84.40</td>
      <td>6.492308</td>
      <td>20.914496</td>
    </tr>
    <tr>
      <th>QB</th>
      <td>81.56</td>
      <td>6.273846</td>
      <td>20.210738</td>
    </tr>
    <tr>
      <th>K</th>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>

