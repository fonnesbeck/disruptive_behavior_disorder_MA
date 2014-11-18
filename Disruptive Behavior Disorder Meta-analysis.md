

    %matplotlib inline
    import matplotlib.pyplot as plt
    import seaborn as sb
    import numpy as np
    import pandas as pd
    import pymc as pm
    from scipy.misc import comb


    np.random.seed(42)

## Data import and cleaning


    study_char = pd.read_excel("DBD Data for Meta Analyses.xlsx", "Study Characteristics", 
                               index_col='RefID', na_values=['-', 'NR'])
    outcomes = pd.read_excel("DBD Data for Meta Analyses.xlsx", "Outcomes", 
                               na_values=['ND', 'NR'])
    demographics = pd.read_excel("DBD Data for Meta Analyses.xlsx", "Pt Demographics", na_values=['-', 'NR'])

Data cleaning


    # Cast outcomes variables to floats
    for col in ('Last FU Mean', 'Last FU SD',):
        outcomes[col] = outcomes[col].astype(float)


    # Recode age category
    study_char['age_cat'] = study_char.AgeCat.replace({'PRE-K':1, 'SCHOOL':0, 'TEEN':2})


    # Fix data label typo
    outcomes['Measure Instrument'] = outcomes['Measure Instrument'].replace({'Eyberg Child Behaviour Inventory, problem Subscale': 
                                            'Eyberg Child Behaviour Inventory, Problem Subscale'})
    outcomes.Units = outcomes.Units.replace({'scale': 'Scale'})


    # Parse followup times and convert to months
    split_fut = outcomes.loc[outcomes['Last FU Time'].notnull(), 'Last FU Time'].apply(lambda x: str(x).split(' ')[:2])
    fut_months = [float(time)/52.*(unit=='weeks') or float(time) for time, unit in split_fut]
    outcomes.loc[outcomes['Last FU Time'].notnull(), 'Last FU Time'] = fut_months

We are assumung all CBC Externalizing values over 50 are T-scores, and those
under 50 are raw scores. This recodes those observations.


    cbce_ind = outcomes['Measure Instrument'].apply(lambda x: x.startswith('Child Behavior Checklist, Externalizing'))
    under_50 = outcomes['BL Mean']<50
    outcomes.loc[cbce_ind & (under_50^True), 'Measure Instrument'] = 'Child Behavior Checklist, Externalizing (T Score)'
    outcomes.loc[cbce_ind & under_50, 'Measure Instrument'] = 'Child Behavior Checklist, Externalizing'

Recode measure instrument variables


    instrument = []
    subtype = []
    units = []
    
    for i,row in outcomes.iterrows():
        separator = row['Measure Instrument'].find(',')
        if separator == -1:
            separator = row['Measure Instrument'].find('-')
        instrument.append(row['Measure Instrument'][:separator])
        s = row['Measure Instrument'][separator+2:]
        paren = s.find('(')
        if paren > -1:
            subtype.append(s[:paren-1])
            units.append(s[paren+1:-1])
        else:
            subtype.append(s)
            if s.endswith('scale'):
                units.append('Scale')
            else:
                units.append('Score')
                
    new_cols = pd.DataFrame({'instrument': instrument, 'subtype': subtype, 
                             'units': units}, index=outcomes.index)


    outcomes['Measure Instrument'].value_counts()




    Eyberg Child Behaviour Inventory, Intensity Subscale            63
    Eyberg Child Behaviour Inventory, Problem Subscale              45
    Child Behavior Checklist, Externalizing (T Score)               33
    Child Behavior Checklist, Externalizing                         11
    Eyberg Child Behaviour Inventory, Intensity Subscale (T Score)    10
    Strengths and Difficulties Questionnaire- Conduct Problems Scale    10
    Child Behavior Checklist, Aggression                             4
    Strengths and Difficulties Questionnaire- Emotional Symptoms Scale     4
    Strengths and Difficulties Questionnaire- Total Difficulties Score     4
    Strengths and Difficulties Questionnaire- Total Score            4
    Eyberg Child Behaviour Inventory, Problem Subscale (T Score)     4
    Strengths and Difficulties Questionnaire- Impact Score           2
    Strengths and Difficulties Questionnaire- Hyperactivity Scale     2
    Child Behavior Checklist, Conduct Problems                       2
    Child Behavior Checklist, Rulebreaking                           2
    Child Behavior Checklist, Conduct Problems (T Score)             2
    dtype: int64




    new_cols.head()




<div style="max-height:1000px;max-width:1500px;overflow:auto;">
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>instrument</th>
      <th>subtype</th>
      <th>units</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td> Eyberg Child Behaviour Inventory</td>
      <td> Intensity Subscale</td>
      <td> T Score</td>
    </tr>
    <tr>
      <th>1</th>
      <td> Eyberg Child Behaviour Inventory</td>
      <td>   Problem Subscale</td>
      <td> T Score</td>
    </tr>
    <tr>
      <th>2</th>
      <td> Eyberg Child Behaviour Inventory</td>
      <td> Intensity Subscale</td>
      <td> T Score</td>
    </tr>
    <tr>
      <th>3</th>
      <td> Eyberg Child Behaviour Inventory</td>
      <td>   Problem Subscale</td>
      <td> T Score</td>
    </tr>
    <tr>
      <th>4</th>
      <td>         Child Behavior Checklist</td>
      <td>      Externalizing</td>
      <td>   Score</td>
    </tr>
  </tbody>
</table>
</div>




    # Append new columns
    outcomes = outcomes.join(new_cols)


    outcomes.intvn.value_counts()




    wlc              41
    tau              37
    iypt             26
    pcit             16
    pppsd             7
    iyptndiyct        6
    mst               5
    it                4
    ppcp              4
    pppo              4
    pmto              3
    pcitc             3
    snap              3
    spokes            3
    pmtndp            3
    pppe              3
    pmtpa             2
    iyct              2
    hncte             2
    setpc             2
    pcitabb           2
    modularndn        2
    pmtsd             2
    hncstd            2
    pmtnds            2
    cpp               1
    cbt               1
    pppstd            1
    coaching          1
    mcfi              1
    modularndcomm     1
    itpt              1
    kitkashrut        1
    sst               1
    pstnds            1
    hnc               1
    hitkashrut        1
    iyptadv           1
    scip              1
    modularndclin     1
    projndsupport     1
    dtype: int64



## Data summaries

Cross-tabulation of the outcome counts by measure instrument


    pd.crosstab(outcomes['instrument'], outcomes['Outcome'])




<div style="max-height:1000px;max-width:1500px;overflow:auto;">
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>Outcome</th>
      <th>01 Behavior, disruptive</th>
      <th>02 Behavior, aggression</th>
      <th>06 Behavior, fighting, destruction, violation</th>
      <th>08 Behavior, other</th>
    </tr>
    <tr>
      <th>instrument</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>Child Behavior Checklist</th>
      <td>  48</td>
      <td> 4</td>
      <td> 2</td>
      <td>  0</td>
    </tr>
    <tr>
      <th>Eyberg Child Behaviour Inventory</th>
      <td> 122</td>
      <td> 0</td>
      <td> 0</td>
      <td>  0</td>
    </tr>
    <tr>
      <th>Strengths and Difficulties Questionnaire</th>
      <td>  10</td>
      <td> 0</td>
      <td> 0</td>
      <td> 16</td>
    </tr>
  </tbody>
</table>
</div>



Distribution of age categories


    study_char.AgeCat.value_counts()




    SCHOOL    46
    PRE-K     26
    TEEN      14
    dtype: int64



Frequencies of various intervention types


    study_char['Intervention Type'].value_counts()




    PHARM                                                           20
    IY-PT                                                            8
    MST                                                              7
    PCIT                                                             5
    IY-PT + IY-CT                                                    4
    BSFT                                                             3
    PMTO                                                             3
    Triple P (enhanced)                                              3
    PT                                                               2
    Fast Track                                                       2
    PCIT-ABB                                                         2
    Triple-P (self-directed)                                         2
    CBT                                                              2
    IY-CT                                                            2
    IY-PT (nurse led)                                                2
    OTH: Intensive treatment                                         2
    PMT (practitioner assisted)                                      1
    HNC                                                              1
    OTH: Modular (nurse administered)                                1
    MF-PEP + TAU                                                     1
    SNAP Under 12 OutReach Project (enhanced)                        1
    OTH: Booster                                                     1
    Coping power                                                     1
    OTH: Child only treatment                                        1
    OTH: Family therapy                                              1
    SCIP (Social cognitive (Dodge's))                                1
    Triple-P (enhanced)                                              1
    PONI                                                             1
    OTH: Parental Stress                                             1
    PCIT (modified)                                                  1
    Triple-P (online)                                                1
    UCPP                                                             1
    IY-PT (brief)                                                    1
    OTH: Modular treatment (community)                               1
    MST (PIT)                                                        1
    OTH: Modular treatment                                           1
    SNAP Under 12 \nOutReach Project(ORP)                            1
    OTH: Instrumental, emotional support & child management skills     1
    IY-PT + IY-CT + IY-TT                                            1
    PMT (perceptive)                                                 1
    PHARM1 + PHARM2                                                  1
    HNC (technology enhanced)                                        1
    OTH: Community Parent Education Program                          1
    Coping Power                                                     1
    Coping power; Coping Power + Booser                              1
    IY-CT + IY-PT                                                    1
    OTH: FFT                                                         1
    OTH: Day Program                                                 1
    OTH: Parents Plus Children's Program                             1
    IY-PT + ADVANCE                                                  1
    OTH: Project support                                             1
    CPS                                                              1
    Coping Power (cultural adaptation)                               1
    SNAP Under 12 OutReach Project                                   1
    Parenting Group (SPOKES)                                         1
    SET-PC                                                           1
    PHARM + PSYCH                                                    1
    Length: 57, dtype: int64



## Extract variables of interest and merge tables


    KQ1 = study_char[study_char.KQ=='KQ1']


    study_varnames = ['Year', 'age_cat', 'Geographic setting', 'Age mean (years) ', 'Age SD (years)', 
                  'Age min (years)', 'Age max (years)', 'Proportion Male (%)']
    
    study_vars = KQ1[study_varnames].rename(columns={'Geographic setting': 'country', 
                               'Age mean (years) ': 'age_mean', 
                               'Age SD (years)': 'age_sd', 
                               'Age min (years)': 'age_min', 
                               'Age max (years)': 'age_max', 
                               'Proportion Male (%)': 'p_male'})


    study_vars.head()




<div style="max-height:1000px;max-width:1500px;overflow:auto;">
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Year</th>
      <th>age_cat</th>
      <th>country</th>
      <th>age_mean</th>
      <th>age_sd</th>
      <th>age_min</th>
      <th>age_max</th>
      <th>p_male</th>
    </tr>
    <tr>
      <th>RefID</th>
      <th></th>
      <th></th>
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
      <th>23 </th>
      <td> 2013</td>
      <td> 1</td>
      <td>         USA</td>
      <td>  2.80</td>
      <td> 0.61</td>
      <td>  2</td>
      <td>  4</td>
      <td> 62</td>
    </tr>
    <tr>
      <th>100</th>
      <td> 2013</td>
      <td> 2</td>
      <td>         USA</td>
      <td> 14.60</td>
      <td> 1.30</td>
      <td> 11</td>
      <td> 18</td>
      <td> 83</td>
    </tr>
    <tr>
      <th>103</th>
      <td> 2013</td>
      <td> 1</td>
      <td>         USA</td>
      <td>  5.67</td>
      <td> 1.72</td>
      <td>  3</td>
      <td>  8</td>
      <td> 53</td>
    </tr>
    <tr>
      <th>141</th>
      <td> 2013</td>
      <td> 0</td>
      <td>         USA</td>
      <td>  9.90</td>
      <td> 1.30</td>
      <td>  8</td>
      <td> 11</td>
      <td> 73</td>
    </tr>
    <tr>
      <th>156</th>
      <td> 2013</td>
      <td> 2</td>
      <td> Netherlands</td>
      <td> 16.00</td>
      <td> 1.31</td>
      <td> 12</td>
      <td> 18</td>
      <td> 73</td>
    </tr>
  </tbody>
</table>
</div>




    study_vars.p_male.hist()




    <matplotlib.axes._subplots.AxesSubplot at 0x1088dcc50>




![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_27_1.png)


Proportion missing


    study_vars.isnull().mean(0).round(2)




    Year        0.00
    age_cat     0.00
    country     0.00
    age_mean    0.13
    age_sd      0.20
    age_min     0.07
    age_max     0.08
    p_male      0.01
    dtype: float64



Will assume the mean age for those which are missing is simply the midpoint
between minimum and maximum values


    est_means = study_vars.apply(lambda x: x.age_min + (x.age_max - x.age_min) / 2, axis=1)[study_vars.age_mean.isnull()]
    study_vars.loc[study_vars.age_mean.isnull(), 'age_mean'] = est_means
    
    study_vars.age_mean.isnull().sum()




    3




    outcomes_varnames = ['Ref ID', 'Measure Instrument', 'instrument', 'subtype', 'units', 
                         'intvn', 'cc', 'pc', 'fc',
                         'BL N', 'BL Mean', 'BL SD', 
                         'EOT \nN', 'EOT Mean', 'EOT \nSD', 'Last FU Time', 'Last FU N', 
                         'Last FU Mean', 'Last FU SD', 'CS Group N', 'CS Mean', 'CS SD']


    outcomes_vars = outcomes[outcomes_varnames].rename(columns={'Ref ID': 'RefID', 
                                                                           'Measure Instrument': 'measure_instrument',
                                                                           'cc': 'child_component',
                                                                           'pc': 'parent_component',
                                                                           'fc': 'family_component',
                                                                           'oc': 'other_component',
                                                                           'BL N': 'baseline_n',
                                                                           'BL Mean': 'baseline_mean',
                                                                           'BL SD': 'baseline_sd', 
                                                                           'EOT \nN': 'end_treat_n', 
                                                                           'EOT Mean': 'end_treat_mean', 
                                                                           'EOT \nSD': 'end_treat_sd', 
                                                                           'Last FU Time': 'followup_time', 
                                                                           'Last FU N': 'followup_n',
                                                                           'Last FU Mean': 'followup_mean', 
                                                                           'Last FU SD': 'followup_sd', 
                                                                           'CS Group N': 'change_n',
                                                                           'CS Mean': 'change_mean',
                                                                           'CS SD': 'change_sd'})

Recode intervention clasification


    control = ((outcomes_vars.child_component^True) & 
               (outcomes_vars.parent_component^True) & 
               (outcomes_vars.family_component^True)).astype(int)
    child_only = ((outcomes_vars.child_component) & 
                  (outcomes_vars.parent_component^True) & 
                  (outcomes_vars.family_component^True)).astype(int)
    parent_only = ((outcomes_vars.child_component^True) & 
                   (outcomes_vars.parent_component) & 
                   (outcomes_vars.family_component^True)).astype(int)
    outcomes_vars.ix[child_only.astype(bool), ['child_component', 'parent_component', 'family_component']]




<div style="max-height:1000px;max-width:1500px;overflow:auto;">
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>child_component</th>
      <th>parent_component</th>
      <th>family_component</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>112</th>
      <td> 1</td>
      <td> 0</td>
      <td> 0</td>
    </tr>
    <tr>
      <th>113</th>
      <td> 1</td>
      <td> 0</td>
      <td> 0</td>
    </tr>
    <tr>
      <th>115</th>
      <td> 1</td>
      <td> 0</td>
      <td> 0</td>
    </tr>
    <tr>
      <th>116</th>
      <td> 1</td>
      <td> 0</td>
      <td> 0</td>
    </tr>
    <tr>
      <th>149</th>
      <td> 1</td>
      <td> 0</td>
      <td> 0</td>
    </tr>
    <tr>
      <th>173</th>
      <td> 1</td>
      <td> 0</td>
      <td> 0</td>
    </tr>
  </tbody>
</table>
</div>




    multi_component = ((parent_only^True) & (child_only^True) & (control^True)).astype(int)
    
    outcomes_vars['child_only'] = child_only
    outcomes_vars['parent_only'] = parent_only
    outcomes_vars['multi_component'] = multi_component

Obtain subset with non-missing EOT data


    eot_subset = outcomes_vars[outcomes_vars.end_treat_mean.notnull() & outcomes_vars.end_treat_sd.notnull()]

Calculate EOT difference


    eot_subset['eot_diff_mean'] = eot_subset.end_treat_mean - eot_subset.baseline_mean

    /usr/local/lib/python3.4/site-packages/IPython/kernel/__main__.py:1: SettingWithCopyWarning: 
    A value is trying to be set on a copy of a slice from a DataFrame.
    Try using .loc[row_indexer,col_indexer] = value instead
    
    See the the caveats in the documentation: http://pandas.pydata.org/pandas-docs/stable/indexing.html#indexing-view-versus-copy
      if __name__ == '__main__':



    eot_subset['eot_diff_sd'] = eot_subset.baseline_sd + eot_subset.end_treat_sd


    eot_subset['eot_diff_n'] = eot_subset[['baseline_n', 'end_treat_n']].min(1)

Distribution of baseline means among outcome metrics


    for instrument in ('Eyberg Child Behaviour Inventory', 
                       'Child Behavior Checklist', 
                       'Strengths and Difficulties Questionnaire'):
        eot_subset[eot_subset.instrument==instrument]['baseline_mean'].hist(by=eot_subset['subtype'], 
                                                                                  sharex=True)
        plt.suptitle(instrument);


![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_44_0.png)



![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_44_1.png)



![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_44_2.png)



    eot_subset.instrument.value_counts()




    Eyberg Child Behaviour Inventory            86
    Child Behavior Checklist                    31
    Strengths and Difficulties Questionnaire    20
    dtype: int64




    eot_subset[eot_subset.RefID==441]




<div style="max-height:1000px;max-width:1500px;overflow:auto;">
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>RefID</th>
      <th>measure_instrument</th>
      <th>instrument</th>
      <th>subtype</th>
      <th>units</th>
      <th>intvn</th>
      <th>child_component</th>
      <th>parent_component</th>
      <th>family_component</th>
      <th>baseline_n</th>
      <th>...</th>
      <th>followup_sd</th>
      <th>change_n</th>
      <th>change_mean</th>
      <th>change_sd</th>
      <th>child_only</th>
      <th>parent_only</th>
      <th>multi_component</th>
      <th>eot_diff_mean</th>
      <th>eot_diff_sd</th>
      <th>eot_diff_n</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>14</th>
      <td> 441</td>
      <td> Eyberg Child Behaviour Inventory, Intensity Su...</td>
      <td> Eyberg Child Behaviour Inventory</td>
      <td> Intensity Subscale</td>
      <td> Scale</td>
      <td> iypt</td>
      <td> 0</td>
      <td> 1</td>
      <td> 0</td>
      <td> 32</td>
      <td>...</td>
      <td>NaN</td>
      <td> NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td> 0</td>
      <td> 1</td>
      <td> 0</td>
      <td>-31.40</td>
      <td> 46.80</td>
      <td> 32</td>
    </tr>
    <tr>
      <th>15</th>
      <td> 441</td>
      <td> Eyberg Child Behaviour Inventory, Problem Subs...</td>
      <td> Eyberg Child Behaviour Inventory</td>
      <td>   Problem Subscale</td>
      <td> Scale</td>
      <td> iypt</td>
      <td> 0</td>
      <td> 1</td>
      <td> 0</td>
      <td> 24</td>
      <td>...</td>
      <td>NaN</td>
      <td> NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td> 0</td>
      <td> 1</td>
      <td> 0</td>
      <td> -9.70</td>
      <td> 12.02</td>
      <td> 24</td>
    </tr>
    <tr>
      <th>16</th>
      <td> 441</td>
      <td> Eyberg Child Behaviour Inventory, Intensity Su...</td>
      <td> Eyberg Child Behaviour Inventory</td>
      <td> Intensity Subscale</td>
      <td> Scale</td>
      <td>  wlc</td>
      <td> 0</td>
      <td> 0</td>
      <td> 0</td>
      <td> 20</td>
      <td>...</td>
      <td>NaN</td>
      <td> NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td> 0</td>
      <td> 0</td>
      <td> 0</td>
      <td> -5.80</td>
      <td> 49.60</td>
      <td> 20</td>
    </tr>
    <tr>
      <th>17</th>
      <td> 441</td>
      <td> Eyberg Child Behaviour Inventory, Problem Subs...</td>
      <td> Eyberg Child Behaviour Inventory</td>
      <td>   Problem Subscale</td>
      <td> Scale</td>
      <td>  wlc</td>
      <td> 0</td>
      <td> 0</td>
      <td> 0</td>
      <td> 17</td>
      <td>...</td>
      <td>NaN</td>
      <td> NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td> 0</td>
      <td> 0</td>
      <td> 0</td>
      <td> -2.88</td>
      <td> 14.59</td>
      <td> 17</td>
    </tr>
  </tbody>
</table>
<p>4 rows × 28 columns</p>
</div>



Several studies use multiple instruments and metrics within instruments


    eot_subset.groupby(['RefID', 'instrument'])['subtype'].value_counts()




    RefID  instrument                                                        
    441    Eyberg Child Behaviour Inventory          Problem Subscale            2
                                                     Intensity Subscale          2
    475    Eyberg Child Behaviour Inventory          Intensity Subscale          2
    539    Child Behavior Checklist                  Externalizing               2
                                                     Aggression                  2
    564    Eyberg Child Behaviour Inventory          Problem Subscale            2
                                                     Intensity Subscale          2
    899    Child Behavior Checklist                  Externalizing               3
           Eyberg Child Behaviour Inventory          Problem Subscale            3
                                                     Intensity Subscale          3
    993    Strengths and Difficulties Questionnaire  Total Difficulties Score    2
                                                     Impact Score                2
                                                     Conduct Problems Scale      2
                                                     Hyperactivity Scale         2
    1236   Eyberg Child Behaviour Inventory          Problem Subscale            6
                                                     Intensity Subscale          6
    1245   Child Behavior Checklist                  Externalizing               2
           Eyberg Child Behaviour Inventory          Intensity Subscale          2
    1511   Child Behavior Checklist                  Externalizing               2
    1585   Eyberg Child Behaviour Inventory          Problem Subscale            2
                                                     Intensity Subscale          2
    1875   Child Behavior Checklist                  Externalizing               3
    1951   Child Behavior Checklist                  Externalizing               2
    2092   Child Behavior Checklist                  Externalizing               3
           Eyberg Child Behaviour Inventory          Intensity Subscale          3
    2117   Child Behavior Checklist                  Externalizing               2
    2219   Eyberg Child Behaviour Inventory          Problem Subscale            3
                                                     Intensity Subscale          3
    2239   Child Behavior Checklist                  Externalizing               2
    2347   Strengths and Difficulties Questionnaire  Total Score                 2
                                                     Emotional Symptoms Scale    2
                                                     Conduct Problems Scale      2
    3211   Eyberg Child Behaviour Inventory          Problem Subscale            2
                                                     Intensity Subscale          2
           Strengths and Difficulties Questionnaire  Total Difficulties Score    2
    3225   Eyberg Child Behaviour Inventory          Problem Subscale            2
                                                     Intensity Subscale          2
    3397   Child Behavior Checklist                  Externalizing               2
    3399   Child Behavior Checklist                  Externalizing               2
    3495   Eyberg Child Behaviour Inventory          Intensity Subscale          4
    3687   Eyberg Child Behaviour Inventory          Problem Subscale            6
                                                     Intensity Subscale          6
    3716   Eyberg Child Behaviour Inventory          Problem Subscale            2
                                                     Intensity Subscale          2
    3766   Child Behavior Checklist                  Conduct Problems            2
           Eyberg Child Behaviour Inventory          Intensity Subscale          5
    3915   Eyberg Child Behaviour Inventory          Problem Subscale            2
                                                     Intensity Subscale          2
    3960   Eyberg Child Behaviour Inventory          Problem Subscale            2
    7109   Eyberg Child Behaviour Inventory          Problem Subscale            2
                                                     Intensity Subscale          2
           Strengths and Difficulties Questionnaire  Emotional Symptoms Scale    2
                                                     Conduct Problems Scale      2
    7723   Child Behavior Checklist                  Externalizing               2
    Length: 54, dtype: int64




    pd.crosstab(eot_subset.instrument, eot_subset.subtype)




<div style="max-height:1000px;max-width:1500px;overflow:auto;">
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>subtype</th>
      <th>Aggression</th>
      <th>Conduct Problems</th>
      <th>Conduct Problems Scale</th>
      <th>Emotional Symptoms Scale</th>
      <th>Externalizing</th>
      <th>Hyperactivity Scale</th>
      <th>Impact Score</th>
      <th>Intensity Subscale</th>
      <th>Problem Subscale</th>
      <th>Total Difficulties Score</th>
      <th>Total Score</th>
    </tr>
    <tr>
      <th>instrument</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
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
      <th>Child Behavior Checklist</th>
      <td> 2</td>
      <td> 2</td>
      <td> 0</td>
      <td> 0</td>
      <td> 27</td>
      <td> 0</td>
      <td> 0</td>
      <td>  0</td>
      <td>  0</td>
      <td> 0</td>
      <td> 0</td>
    </tr>
    <tr>
      <th>Eyberg Child Behaviour Inventory</th>
      <td> 0</td>
      <td> 0</td>
      <td> 0</td>
      <td> 0</td>
      <td>  0</td>
      <td> 0</td>
      <td> 0</td>
      <td> 50</td>
      <td> 36</td>
      <td> 0</td>
      <td> 0</td>
    </tr>
    <tr>
      <th>Strengths and Difficulties Questionnaire</th>
      <td> 0</td>
      <td> 0</td>
      <td> 6</td>
      <td> 4</td>
      <td>  0</td>
      <td> 2</td>
      <td> 2</td>
      <td>  0</td>
      <td>  0</td>
      <td> 4</td>
      <td> 2</td>
    </tr>
  </tbody>
</table>
</div>




    x = eot_subset[eot_subset.instrument=='Eyberg Child Behaviour Inventory']
    pd.crosstab(x.instrument, x.subtype)




<div style="max-height:1000px;max-width:1500px;overflow:auto;">
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>subtype</th>
      <th>Intensity Subscale</th>
      <th>Problem Subscale</th>
    </tr>
    <tr>
      <th>instrument</th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>Eyberg Child Behaviour Inventory</th>
      <td> 50</td>
      <td> 36</td>
    </tr>
  </tbody>
</table>
</div>




    x = eot_subset[eot_subset.instrument=='Child Behavior Checklist']
    pd.crosstab(x.instrument, x.subtype)




<div style="max-height:1000px;max-width:1500px;overflow:auto;">
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>subtype</th>
      <th>Aggression</th>
      <th>Conduct Problems</th>
      <th>Externalizing</th>
    </tr>
    <tr>
      <th>instrument</th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>Child Behavior Checklist</th>
      <td> 2</td>
      <td> 2</td>
      <td> 27</td>
    </tr>
  </tbody>
</table>
</div>




    x = eot_subset[eot_subset.instrument=='Strengths and Difficulties Questionnaire']
    pd.crosstab(x.instrument, x.subtype)




<div style="max-height:1000px;max-width:1500px;overflow:auto;">
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>subtype</th>
      <th>Conduct Problems Scale</th>
      <th>Emotional Symptoms Scale</th>
      <th>Hyperactivity Scale</th>
      <th>Impact Score</th>
      <th>Total Difficulties Score</th>
      <th>Total Score</th>
    </tr>
    <tr>
      <th>instrument</th>
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
      <th>Strengths and Difficulties Questionnaire</th>
      <td> 6</td>
      <td> 4</td>
      <td> 2</td>
      <td> 2</td>
      <td> 4</td>
      <td> 2</td>
    </tr>
  </tbody>
</table>
</div>



Merge study variables and outcomes


    merged_vars = study_vars.merge(eot_subset, left_index=True, right_on='RefID')
    merged_vars.shape




    (137, 36)



For now, restrict to the three most prevalent metrics.


    merged_vars.measure_instrument.value_counts()




    Eyberg Child Behaviour Inventory, Intensity Subscale            46
    Eyberg Child Behaviour Inventory, Problem Subscale              36
    Child Behavior Checklist, Externalizing (T Score)               20
    Child Behavior Checklist, Externalizing                          7
    Strengths and Difficulties Questionnaire- Conduct Problems Scale     6
    Strengths and Difficulties Questionnaire- Emotional Symptoms Scale     4
    Strengths and Difficulties Questionnaire- Total Difficulties Score     4
    Eyberg Child Behaviour Inventory, Intensity Subscale (T Score)     4
    Child Behavior Checklist, Aggression                             2
    Strengths and Difficulties Questionnaire- Total Score            2
    Strengths and Difficulties Questionnaire- Hyperactivity Scale     2
    Child Behavior Checklist, Conduct Problems (T Score)             2
    Strengths and Difficulties Questionnaire- Impact Score           2
    dtype: int64




    analysis_subset = merged_vars[merged_vars.measure_instrument.isin(merged_vars.measure_instrument.value_counts().index[:4])]
    analysis_subset.groupby('measure_instrument')['baseline_mean'].max()




    measure_instrument
    Child Behavior Checklist, Externalizing                  30.90
    Child Behavior Checklist, Externalizing (T Score)        77.10
    Eyberg Child Behaviour Inventory, Intensity Subscale    186.44
    Eyberg Child Behaviour Inventory, Problem Subscale       28.62
    Name: baseline_mean, dtype: float64




    analysis_subset.groupby('measure_instrument').baseline_mean.hist()




    measure_instrument
    Child Behavior Checklist, Externalizing                 Axes(0.125,0.125;0.775x0.775)
    Child Behavior Checklist, Externalizing (T Score)       Axes(0.125,0.125;0.775x0.775)
    Eyberg Child Behaviour Inventory, Intensity Subscale    Axes(0.125,0.125;0.775x0.775)
    Eyberg Child Behaviour Inventory, Problem Subscale      Axes(0.125,0.125;0.775x0.775)
    Name: baseline_mean, dtype: object




![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_58_1.png)



    analysis_subset['baseline_mean'].hist(by=analysis_subset['measure_instrument'],sharex=True)




    array([[<matplotlib.axes._subplots.AxesSubplot object at 0x1091950b8>,
            <matplotlib.axes._subplots.AxesSubplot object at 0x108d18da0>],
           [<matplotlib.axes._subplots.AxesSubplot object at 0x108bab710>,
            <matplotlib.axes._subplots.AxesSubplot object at 0x109028c18>]], dtype=object)




![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_59_1.png)



    analysis_subset['eot_diff_mean'].hist(by=analysis_subset['measure_instrument'],sharex=True)




    array([[<matplotlib.axes._subplots.AxesSubplot object at 0x108b08a90>,
            <matplotlib.axes._subplots.AxesSubplot object at 0x108753550>],
           [<matplotlib.axes._subplots.AxesSubplot object at 0x1089026a0>,
            <matplotlib.axes._subplots.AxesSubplot object at 0x108de6a90>]], dtype=object)




![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_60_1.png)



    for x in analysis_subset.measure_instrument.unique():
        plt.figure()
        analysis_subset[analysis_subset.measure_instrument==x].baseline_mean.plot(kind='kde', alpha=0.4, grid=False)
        analysis_subset[analysis_subset.measure_instrument==x].end_treat_mean.plot(kind='kde', alpha=0.4, grid=False)
        plt.gca().set_xlabel(x)


![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_61_0.png)



![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_61_1.png)



![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_61_2.png)



![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_61_3.png)


## Meta-analysis

Number of studies in analysis subset


    unique_studies = analysis_subset.RefID.unique().tolist()
    len(unique_studies)




    26



We are restricting the analysis to the 4 most prevalent measure instruments in
the database.


    unique_measures = analysis_subset.measure_instrument.unique().tolist()
    k = len(unique_measures)
    unique_measures




    ['Eyberg Child Behaviour Inventory, Intensity Subscale',
     'Eyberg Child Behaviour Inventory, Problem Subscale',
     'Child Behavior Checklist, Externalizing (T Score)',
     'Child Behavior Checklist, Externalizing']



Three intervention components were coded:

* `child_component`
* `parent_component`
* `multi_component`


    analysis_subset[['child_only', 'parent_only', 'multi_component']].sum(0)




    child_only          6
    parent_only        33
    multi_component    27
    dtype: int64




    p_male, age_cat, intvn = analysis_subset[['p_male', 'age_cat', 'intvn']].values.T
    child_only, parent_only, multi_component = analysis_subset[['child_only', 'parent_only', 
                                                                           'multi_component']].values.T
    
    change_n, change_mean, change_sd = analysis_subset[['eot_diff_n', 'eot_diff_mean', 'eot_diff_sd']].values.T


    school = (analysis_subset.age_cat.values==0).astype(int)
    pre_k = (analysis_subset.age_cat.values==1).astype(int)
    teen = (analysis_subset.age_cat.values==2).astype(int)

The response variable is a multivariate normal of dimension k=4, for each of the
measure instruments.

$$\left({
\begin{array}{c}
  {m_1}\\
  {m_2}\\
  {m_3}\\
  {m_4}
\end{array}
}\right)_i \sim\text{MVN}(\mathbf{\mu},\Sigma)$$

Means for each study are a draw from a multivariate normal.


    wishart = True
    
    mu = pm.Normal('mu', 0, 0.001, value=[0]*k)
    
    if wishart:
        T = pm.Wishart('T', k, np.eye(k), value=np.eye(k))
        
        m = [pm.MvNormal('m_{}'.format(i), mu, T, value=[0]*k) for i in range(len(unique_studies))]
    else:
        sigmas = pm.Uniform('sigmas', 0, 100, value=[10]*k)
        rhos = pm.Uniform('rhos', -1, 1, value=[0]*int(comb(k, 2)))
    
        Sigma = pm.Lambda('Sigma', lambda s=sigmas, r=rhos: np.array([[s[0]**2, s[0]*s[1]*r[0], s[0]*s[2]*r[1], s[0]*s[3]*r[2]],
                                                             [s[0]*s[1]*r[0], s[1]**2, s[1]*s[2]*r[3], s[1]*s[3]*r[4]],
                                                             [s[0]*s[2]*r[1], s[1]*s[2]*r[3], s[2]**2, s[2]*s[3]*r[5]],
                                                             [s[0]*s[3]*r[2], s[1]*s[3]*r[4], s[2]*s[3]*r[5], s[3]**2]]))
        
        m = [pm.MvNormalCov('m_{}'.format(i), mu, Sigma, value=[0]*k) for i in range(len(unique_studies))]
    


Unique intervention labels for each component; we will use these for component
random effects.


    unique_child_intvn = np.unique(intvn[child_only.astype(bool)]).tolist()
    unique_parent_intvn = np.unique(intvn[parent_only.astype(bool)]).tolist()
    unique_multi_intvn = np.unique(intvn[multi_component.astype(bool)]).tolist()


    # Indices to random effect labels
    child_component_index = [unique_child_intvn.index(x) for x in intvn[child_only.astype(bool)]]
    parent_component_index = [unique_parent_intvn.index(x) for x in intvn[parent_only.astype(bool)]]
    multi_component_index = [unique_multi_intvn.index(x) for x in intvn[multi_component.astype(bool)]]

Treatment component random effects

$$X_i = \left[{
\begin{array}{c}
  {x_c}\\
  {x_p}\\
  {x_f}\\
\end{array}
}\right]_i$$

$$\begin{aligned}
\beta_j^{(c)} &\sim N(\mu_{\beta}^{(c)},\tau_{\beta}^{(c)}) \\
\beta_j^{(p)} &\sim N(\mu_{\beta}^{(p)},\tau_{\beta}^{(p)}) \\
\beta_j^{(f)} &\sim N(\mu_{\beta}^{(f)},\tau_{\beta}^{(f)})
\end{aligned}$$


    mu_beta = pm.Normal('mu_beta', 0, 0.001, value=[0]*3)
    # sig_beta = pm.Uniform('sig_beta', 0, 100, value=1)
    # tau_beta = sig_beta ** -2
    tau_beta = pm.Gamma('tau_beta', 1, 0.1, value=1)
    
    beta_c = pm.Normal('beta_c', mu_beta[0], tau_beta, value=[0]*len(unique_child_intvn))
    beta_p = pm.Normal('beta_p', mu_beta[1], tau_beta, value=[0]*len(unique_parent_intvn))
    beta_m = pm.Normal('beta_m', mu_beta[2], tau_beta, value=[0]*len(unique_multi_intvn))
    
    b_c = pm.Lambda('b_c', lambda b=beta_c: 
                 np.array([b[unique_child_intvn.index(x)] if child_only[i] else 0 for i,x in enumerate(intvn)]))
    b_p = pm.Lambda('b_p', lambda b=beta_p: 
                 np.array([b[unique_parent_intvn.index(x)] if parent_only[i] else 0 for i,x in enumerate(intvn)]))
    b_m = pm.Lambda('b_m', lambda b=beta_m: 
                 np.array([b[unique_multi_intvn.index(x)] if multi_component[i] else 0 for i,x in enumerate(intvn)]))



    best = pm.Lambda('best', lambda b=mu_beta:  (b==b.min()).astype(int))

Interaction of parent and multi-component with pre-k children.


    interaction = False
    
    if interaction:
        beta_pk_p = pm.Normal('beta_pk_p', 0, 1e-5, value=0)
        beta_pk_m = pm.Normal('beta_pk_m', 0, 1e-5, value=0)
        b_pk_p = pm.Lambda('b_pk_p', lambda b=beta_pk_p: b * parent_only * pre_k)
        b_pk_m = pm.Lambda('b_pk_m', lambda b=beta_pk_m: b * multi_component * pre_k)


    betas = b_c + b_p + b_m 
    
    if interaction:
        betas = betas + b_pk_p + b_pk_m

Covariate effects of age and percent female.

$$\alpha \sim N(0, 1e5)$$


    alpha_age = pm.Normal('alpha_age', 0, 1e-5, value=[1,2])

Unique study ID (`RefID`) and measure ID (`measure_instrument`) values.


    study_id = [unique_studies.index(x) for x in analysis_subset.RefID]
    measure_id = [unique_measures.index(x) for x in analysis_subset.measure_instrument]

Calculate expected response (treatment difference) as a function of treatment
and covariates.

$$\theta_i = m_{j[i]k} + X_i \beta + \alpha x_{age}$$


    baseline_sd = analysis_subset.baseline_sd.values
    
    @pm.deterministic
    def theta(m=m, betas=betas, alpha_age=alpha_age):  
    
        mi = [m[i][j] for i,j in zip(study_id, measure_id)]
        
        age_effect = np.array([alpha_age[a-1] if a else 0 for a in age_cat])
        
        return(mi + baseline_sd*(betas + age_effect))

Expected treatment effect for pre-K undergoing multi-component intervention,
measused by Eyberg Child Behaviour Inventory, Intensity Subscale


    baseline = pm.MvNormalCov('baseline', mu, T, value=[0]*k)


    ecbi_intensity_sd = baseline_sd[np.array(measure_id)==0].mean()
    
    prek_intensity_pred = pm.Lambda('prek_intensity_pred', 
                                lambda mu=baseline, a=alpha_age, b=mu_beta: mu[0] + ecbi_intensity_sd*(b + a[0]) )
    school_intensity_pred = pm.Lambda('school_intensity_pred', 
                                lambda mu=baseline, a=alpha_age, b=mu_beta: mu[0] + ecbi_intensity_sd*b )
    teen_intensity_pred = pm.Lambda('teen_intensity_pred', 
                                lambda mu=baseline, a=alpha_age, b=mu_beta: mu[0] + ecbi_intensity_sd*(b + a[1]) )
    
    ecbi_problem_sd = baseline_sd[np.array(measure_id)==1].mean()
    
    prek_problem_pred = pm.Lambda('prek_problem_pred', 
                                lambda mu=baseline, a=alpha_age, b=mu_beta: mu[1] + ecbi_problem_sd*(b + a[0]) )
    school_problem_pred = pm.Lambda('school_problem_pred', 
                                lambda mu=baseline, a=alpha_age, b=mu_beta: mu[1] + ecbi_problem_sd*b )
    teen_problem_pred = pm.Lambda('teen_problem_pred', 
                                lambda mu=baseline, a=alpha_age, b=mu_beta: mu[1] + ecbi_problem_sd*(b + a[1]) )
    
    cbct_sd = baseline_sd[np.array(measure_id)==2].mean()
    
    prek_tscore_pred = pm.Lambda('prek_tscore_pred', 
                                lambda mu=baseline, a=alpha_age, b=mu_beta: mu[2] + cbct_sd*(b + a[0]) )
    school_tscore_pred = pm.Lambda('school_tscore_pred', 
                                lambda mu=baseline, b=mu_beta: mu[2] + cbct_sd*b )
    teen_tscore_pred = pm.Lambda('teen_tscore_pred', 
                                lambda mu=baseline, a=alpha_age, b=mu_beta: mu[2] + cbct_sd*(b + a[1]) )
    
    cbcr_sd = baseline_sd[np.array(measure_id)==3].mean()
    
    prek_raw_pred = pm.Lambda('prek_raw_pred', 
                                lambda mu=baseline, a=alpha_age, b=mu_beta: mu[3] + cbcr_sd*(b + a[0]) )
    school_raw_pred = pm.Lambda('school_raw_pred', 
                                lambda mu=baseline, b=mu_beta: mu[3] + cbcr_sd*b )
    teen_raw_pred = pm.Lambda('teen_raw_pred', 
                                lambda mu=baseline, a=alpha_age, b=mu_beta: mu[3] + cbcr_sd*(b + a[1]) )

Finally, the likelihood is just a normal distribution, with the observed
standard error of the treatment effect as the standard deviation of the
estimates.

$$d_i \sim N(\theta_i, \hat{\sigma}^2)$$


    change_se = change_sd/np.sqrt(change_n)


    d = pm.Normal('d', theta, change_se**-2, observed=True, value=change_mean)

Posterior predictive samples


    d_sim = pm.Normal('d_sim', theta, change_se**-2, size=len(change_mean))


    import appnope
    appnope.nope()
    
    M = pm.MCMC(locals())
    M.use_step_method(pm.AdaptiveMetropolis, [mu])
    M.use_step_method(pm.AdaptiveMetropolis, m)
    M.use_step_method(pm.AdaptiveMetropolis, mu_beta)
    M.use_step_method(pm.AdaptiveMetropolis, [beta_c, beta_p, beta_m])


    M.sample(200000, 190000)

     [-----------------100%-----------------] 200000 of 200000 complete in 585.7 sec

Summary of estimates of intervention components


    pm.Matplot.summary_plot([mu_beta], custom_labels=['Child component', 'Parent component', 
                                                                  'Multi-component'])

    Could not calculate Gelman-Rubin statistics. Requires multiple chains of equal length.



![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_100_1.png)



    mu_beta.summary()

    
    mu_beta:
     
    	Mean             SD               MC Error        95% HPD interval
    	------------------------------------------------------------------
    	-0.56            0.3              0.016            [-1.191 -0.004]
    	-1.007           0.217            0.011            [-1.418 -0.564]
    	-1.143           0.247            0.012            [-1.607 -0.629]
    	
    	
    	Posterior quantiles:
    	
    	2.5             25              50              75             97.5
    	 |---------------|===============|===============|---------------|
    	-1.165           -0.748          -0.558         -0.364        0.036
    	-1.418           -1.155          -1.008         -0.868        -0.563
    	-1.65            -1.302          -1.149         -0.989        -0.654
    	



    best.summary()

    
    best:
     
    	Mean             SD               MC Error        95% HPD interval
    	------------------------------------------------------------------
    	0.034            0.182            0.005                  [ 0.  0.]
    	0.322            0.467            0.021                  [ 0.  1.]
    	0.644            0.479            0.02                   [ 0.  1.]
    	
    	
    	Posterior quantiles:
    	
    	2.5             25              50              75             97.5
    	 |---------------|===============|===============|---------------|
    	0.0              0.0             0.0            0.0           1.0
    	0.0              0.0             0.0            1.0           1.0
    	0.0              0.0             1.0            1.0           1.0
    	



    pm.Matplot.plot(mu_beta)

    Plotting mu_beta_0
    Plotting mu_beta_1
    Plotting mu_beta_2



![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_103_1.png)



    if interaction:
        pm.Matplot.summary_plot([beta_pk_m, beta_pk_p])

Difference means by measure instrument.


    plt.figure(figsize=(24,4))
    pm.Matplot.summary_plot([mu], custom_labels=unique_measures)

    Could not calculate Gelman-Rubin statistics. Requires multiple chains of equal length.



![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_106_1.png)



    pm.Matplot.plot(mu)

    Plotting mu_0
    Plotting mu_1
    Plotting mu_2
    Plotting mu_3



![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_107_1.png)



    if not wishart:
        plt.figure(figsize=(24,4))
        pm.Matplot.summary_plot([sigmas], custom_labels=unique_measures)


    if not wishart:
        pm.Matplot.summary_plot([rhos], custom_labels=['rho12', 'rho13', 'rho14', 'rho23', 'rho24', 'rho34'])

Age effects for pre-k (top) and teen (bottom) groups, relative to pre-teen.


    pm.Matplot.summary_plot([alpha_age], custom_labels=['pre-K', 'teen'])

    Could not calculate Gelman-Rubin statistics. Requires multiple chains of equal length.



![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_111_1.png)



    alpha_age.summary()

    
    alpha_age:
     
    	Mean             SD               MC Error        95% HPD interval
    	------------------------------------------------------------------
    	-0.241           0.072            0.005            [-0.372 -0.09 ]
    	-0.149           0.145            0.011            [-0.436  0.12 ]
    	
    	
    	Posterior quantiles:
    	
    	2.5             25              50              75             97.5
    	 |---------------|===============|===============|---------------|
    	-0.379           -0.288          -0.242         -0.192        -0.097
    	-0.436           -0.251          -0.145         -0.044        0.12
    	



    pm.Matplot.plot(alpha_age)

    Plotting alpha_age_0
    Plotting alpha_age_1



![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_113_1.png)


## Outcome Plots


    traces = [[school_intensity_pred, prek_intensity_pred, teen_intensity_pred],
              [school_problem_pred, prek_problem_pred, teen_problem_pred],
              [school_tscore_pred, prek_tscore_pred, teen_tscore_pred],
              [school_raw_pred, prek_raw_pred, teen_raw_pred]]


    sb.set(style="white", palette="hot")
    sb.despine(left=True)
    
    #colors = '#fef0d9','#fdcc8a','#fc8d59','#d7301f'
    colors = sb.cubehelix_palette(4, start=2, rot=0, dark=.25, light=.75, reverse=False)
    
    titles = ['School Children', 'Pre-K Children', 'Teenage Children']
    
    for i,measure in enumerate(unique_measures):
        
        measure_traces = traces[i]
        
        for j, trace in enumerate(measure_traces):
            
            x = np.random.choice(analysis_subset[analysis_subset.measure_instrument==measure].baseline_mean, 10000)
            
            c1, p1, m1 = trace.trace().T
            
            plt.figure()
            g = sb.distplot(x + c1, color=colors[0])
            sb.distplot(x + p1, color=colors[1])
            sb.distplot(x + m1, color=colors[2])
            if j:
                age_effect = alpha_age.trace()[:, j-1]
            else:
                age_effect = 0
            sb.distplot(x + baseline.trace()[:, i] + age_effect, color=colors[3]);
            g.set_title(titles[j])
    
            g.legend(g.lines, ['Child-only', 'Parent-only', 'Multi-component', 'Control/TAU'])
    
            g.set_xlabel(measure)


    <matplotlib.figure.Figure at 0x108a7e4a8>



![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_116_1.png)



![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_116_2.png)



![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_116_3.png)



![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_116_4.png)



![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_116_5.png)



![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_116_6.png)



![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_116_7.png)



![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_116_8.png)



![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_116_9.png)



![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_116_10.png)



![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_116_11.png)



![png](Disruptive%20Behavior%20Disorder%20Meta-analysis_files/Disruptive%20Behavior%20Disorder%20Meta-analysis_116_12.png)



    x1 = np.random.choice(analysis_subset[analysis_subset.measure_instrument==unique_measures[0]].baseline_mean, 10000)


    #pm.Matplot.gof_plot(d_sim, change_mean, verbose=0)
