---
layout: default
title: eventstudyinteract
parent: Stata code
nav_order: 6
mathjax: true
image: "../../../assets/images/DiD.png"
---

# eventstudyinteract (Sun and Abraham 2020)
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Introduction

The *eventstudyinteract* command is written by Liyang Sun based on the Sun and Abraham 2020 paper [Estimating Dynamic Treatment Effects in Event Studies with Heterogeneous Treatment Effects](https://www.sciencedirect.com/science/article/pii/S030440762030378X).

The command potentially has some issues. See the code and the graphs below for details. If you have some insights on how to fix this, then please email me, or post in the [issues](https://github.com/asjadnaqvi/DiD/issues) section on GitHub.

## Installation and options

Install the command from SSC:

```applescript
ssc install eventstudyinteract, replace
```

Take a look at the help file:

```applescript
help eventstudyinteract
```

The core syntax is as follows:

```applescript
eventstudyinteract Y *lags* *leads*, vce(cluster *var*) absorb(*i* *t*) cohort(first_treat) control_cohort(*variable*)
```

where: 

| Variable | Description |
| ----- | ----- |
| Y | outcome variable |
| i | panel id |
| t | time variable  |
| *lags* | manually generated lag variables  |
| *leads* | manually generated lead variables  |
| first_treat | timing of first treatment (missing for untreated groups) |
| control_cohort(*var*) | The variable here is either never treated observations, or last treated cohorts  |


### The options

| Option | Description |


*INCOMPLETE*


## Generate sample data


Here we generate a test dataset with heterogeneous treatments:

```applescript
clear

local units = 30
local start = 1
local end 	= 60

local time = `end' - `start' + 1
local obsv = `units' * `time'
set obs `obsv'

egen id	   = seq(), b(`time')  
egen t 	   = seq(), f(`start') t(`end') 	

sort  id t
xtset id t


set seed 20211222

gen Y 	   		= 0		// outcome variable	
gen D 	   		= 0		// intervention variable
gen cohort      = .  	// treatment cohort
gen effect      = .		// treatment effect size
gen first_treat = .		// when the treatment happens for each cohort
gen rel_time	= .     // time - first_treat

levelsof id, local(lvls)
foreach x of local lvls {
	local chrt = runiformint(0,5)	
	replace cohort = `chrt' if id==`x'
}


levelsof cohort , local(lvls)  
foreach x of local lvls {
	
	local eff = runiformint(2,10)
		replace effect = `eff' if cohort==`x'
			
	local timing = runiformint(`start',`end' + 20)	// 
	replace first_treat = `timing' if cohort==`x'
	replace first_treat = . if first_treat > `end'
		replace D = 1 if cohort==`x' & t>= `timing' 
}

replace rel_time = t - first_treat
replace Y = id + t + cond(D==1, effect * rel_time, 0) + rnormal()
```

Generate the graph:


```applescript
xtline Y, overlay legend(off)
```

<img src="../../../assets/images/test_data.png" height="300">

## Test the command


For `eventstudyinteract` we need to generate 10 leads and lags and drop the first lead:

```applescript
summ rel_time
local relmin = abs(r(min))
local relmax = abs(r(max))

	// leads
	cap drop F_*
	forval x = 2/`relmin' {  // drop the first lead
		gen F_`x' = rel_time == -`x'
	}

	
	//lags
	cap drop L_*
	forval x = 0/`relmax' {
		gen L_`x' = rel_time ==  `x'
	}
```

and we need to generate the `control_cohort` variables. Here we generate the `never_treated` and `last_cohort` variables as suggested in the help file:

```
gen never_treat = first_treat==.

sum first_treat
gen last_cohort = first_treat==r(max) // dummy for the latest- or never-treated cohort
```



Let's try the basic `eventstudyinteract` command the never_treated as the `control_cohort`:

```applescript
eventstudyinteract Y L_* F_*, vce(cluster id) absorb(id t) cohort(first_treat) control_cohort(never_treat)
```


which will show this output:

```xml
IW estimates for dynamic effects                        Number of obs =  1,800
Absorbing 2 HDFE groups                                 F(236, 29)    =      .
                                                        Prob > F      =      .
                                                        R-squared     = 0.9999
                                                        Adj R-squared = 0.9999
                                                        Root MSE      = 1.0114
                                    (Std. err. adjusted for 30 clusters in id)
------------------------------------------------------------------------------
             |               Robust
           Y | Coefficient  std. err.      t    P>|t|     [95% conf. interval]
-------------+----------------------------------------------------------------
         L_0 |  -.0705566   .4520534    -0.16   0.877    -.9951096    .8539963
         L_1 |   8.477542   .4379319    19.36   0.000     7.581871    9.373214
         L_2 |   17.69445   .6109211    28.96   0.000     16.44497    18.94392
         L_3 |   25.97508   .6262269    41.48   0.000      24.6943    27.25585
         L_4 |   34.74591   .9050206    38.39   0.000     32.89493    36.59688
         L_5 |    42.5053   1.223893    34.73   0.000     40.00216    45.00845
         L_6 |   51.87653   1.498852    34.61   0.000     48.81103    54.94202
         L_7 |   60.49124   1.799291    33.62   0.000     56.81128     64.1712
         L_8 |    69.1176   1.982205    34.87   0.000     65.06353    73.17167
         L_9 |   77.28842   2.229958    34.66   0.000     72.72764     81.8492
        L_10 |   85.61498   2.555366    33.50   0.000     80.38867    90.84129
        L_11 |   93.99587   2.708476    34.70   0.000     88.45642    99.53533
        L_12 |   103.5575   3.047215    33.98   0.000     97.32524    109.7897
        L_13 |   110.9889   3.357933    33.05   0.000     104.1212    117.8567
        L_14 |    119.482   3.757873    31.80   0.000     111.7963    127.1677
        L_15 |    128.809   3.858744    33.38   0.000     120.9169     136.701
        L_16 |   137.5689    4.21657    32.63   0.000     128.9451    146.1928
        L_17 |   146.0436   4.597805    31.76   0.000       136.64    155.4471
        L_18 |   154.8424   4.586774    33.76   0.000     145.4614    164.2234
        L_19 |   163.6468   4.861225    33.66   0.000     153.7044    173.5891
        L_20 |   171.9099   5.064144    33.95   0.000     161.5526    182.2673
        L_21 |   179.6032   5.309208    33.83   0.000     168.7447    190.4618
        L_22 |   189.3146   5.658889    33.45   0.000     177.7409    200.8883
        L_23 |   203.2878   5.643981    36.02   0.000     191.7445     214.831
        L_24 |   212.1516   5.841365    36.32   0.000     200.2046    224.0985
        L_25 |   221.2972   6.233885    35.50   0.000     208.5475    234.0469
        L_26 |     230.77   6.181821    37.33   0.000     218.1268    243.4132
        L_27 |    268.352   .9300789   288.53   0.000     266.4497    270.2542
        L_28 |   279.7236   .8413134   332.48   0.000     278.0029    281.4442
        L_29 |   289.9827   .7938728   365.28   0.000      288.359    291.6063
        L_30 |   299.9447   1.061719   282.51   0.000     297.7733    302.1162
        L_31 |   308.1538   .7205883   427.64   0.000       306.68    309.6276
        L_32 |   320.2466   .6393276   500.91   0.000      318.939    321.5542
        L_33 |    329.467   .7404276   444.97   0.000     327.9527    330.9814
        L_34 |   340.2075   .8140191   417.94   0.000     338.5426    341.8723
        L_35 |   349.2469   .6694048   521.73   0.000     347.8778    350.6159
        L_36 |   359.3307   .8590308   418.30   0.000     357.5737    361.0876
         F_2 |   .2679339   .5145674     0.52   0.607    -.7844745    1.320342
         F_3 |   .1303887   .3615731     0.36   0.721    -.6091113    .8698886
         F_4 |  -.1227137   .4505851    -0.27   0.787    -1.044264    .7988363
         F_5 |  -.3586946    .483613    -0.74   0.464    -1.347794    .6304051
         F_6 |  -.4161922   .4326221    -0.96   0.344    -1.301004    .4686194
         F_7 |  -.3196142   .4544008    -0.70   0.487    -1.248968    .6097397
         F_8 |  -.0626867   .4604898    -0.14   0.893    -1.004494    .8791206
         F_9 |  -.0926511   .4296329    -0.22   0.831    -.9713491    .7860469
        F_10 |   .1102947   .4471147     0.25   0.807    -.8041576    1.024747
        F_11 |  -.2264116   .4586641    -0.49   0.625    -1.164485    .7116618
        F_12 |  -.1182709    .519532    -0.23   0.822    -1.180833    .9442913
        F_13 |   .1261863   .5312998     0.24   0.814    -.9604437    1.212816
        F_14 |   .1467978   .3339677     0.44   0.664    -.5362428    .8298383
        F_15 |    .286046   .5164297     0.55   0.584    -.7701712    1.342263
        F_16 |   .1889184     .56528     0.33   0.741    -.9672089    1.345046
        F_17 |  -.7257636   .3417224    -2.12   0.042    -1.424664   -.0268629
        F_18 |  -.0111467   .4849795    -0.02   0.982    -1.003041    .9807478
        F_19 |  -.4036042   .3890966    -1.04   0.308    -1.199396    .3921876
        F_20 |    .083998   .3974126     0.21   0.834    -.7288021     .896798
        F_21 |   .0450013   .3697536     0.12   0.904    -.7112297    .8012323
        F_22 |  -.5576153   .4764867    -1.17   0.251     -1.53214    .4169094
        F_23 |   .1384066   .4293158     0.32   0.749    -.7396429    1.016456
        F_24 |   .9187571   .5323275     1.73   0.095    -.1699749    2.007489
        F_25 |  -.1651529   .5467614    -0.30   0.765    -1.283406    .9530998
        F_26 |   .0420085   .4311221     0.10   0.923    -.8397351    .9237522
        F_27 |  -.2333004   .5322703    -0.44   0.664    -1.321915    .8553146
        F_28 |  -.2905799   .4986613    -0.58   0.565    -1.310457     .729297
        F_29 |    .524472   .5439824     0.96   0.343    -.5880969    1.637041
        F_30 |   .4916469    .596472     0.82   0.417    -.7282752    1.711569
        F_31 |   .1394772   .5753502     0.24   0.810    -1.037246      1.3162
        F_32 |   .2195107   .4553865     0.48   0.633    -.7118592    1.150881
        F_33 |    .433028   .5843814     0.74   0.465    -.7621663    1.628222
        F_34 |   .0184928   .8632068     0.02   0.983    -1.746963    1.783949
        F_35 |    .106708   .5945981     0.18   0.859    -1.109382    1.322798
        F_36 |   .1068798   .7019857     0.15   0.880    -1.328842    1.542602
        F_37 |   .0648712   .6159508     0.11   0.917     -1.19489    1.324632
        F_38 |  -.8113278   1.236346    -0.66   0.517     -3.33994    1.717284
        F_39 |  -.5387131   .9568228    -0.56   0.578    -2.495636    1.418209
        F_40 |  -.7248558   1.007424    -0.72   0.478     -2.78527    1.335558
        F_41 |  -.2235736   .5398104    -0.41   0.682     -1.32761    .8804625
        F_42 |  -.3152135   1.014359    -0.31   0.758    -2.389811    1.759384
        F_43 |   .5507069   .8185869     0.67   0.506    -1.123491    2.224905
        F_44 |   .4711243   .7455523     0.63   0.532    -1.053701     1.99595
        F_45 |  -1.302809   .9750776    -1.34   0.192    -3.297066    .6914491
        F_46 |  -.1432933   .5311437    -0.27   0.789    -1.229604    .9430176
        F_47 |  -.5297326   1.077463    -0.49   0.627    -2.733392    1.673927
        F_48 |  -.3128089    .743813    -0.42   0.677    -1.834077    1.208459
        F_49 |  -.0590544   .7666954    -0.08   0.939    -1.627123    1.509014
        F_50 |  -1.156447   .3122062    -3.70   0.001     -1.79498   -.5179134
        F_51 |  -.2063065   .7599222    -0.27   0.788    -1.760522    1.347909
        F_52 |   .0657318   .5711765     0.12   0.909    -1.102455    1.233919
        F_53 |  -.5740659   .5549477    -1.03   0.309    -1.709061    .5609296
        F_54 |   .3670635   1.075711     0.34   0.735    -1.833013     2.56714
        F_55 |  -.4353981   .7411865    -0.59   0.561    -1.951295    1.080498
------------------------------------------------------------------------------
```


In order to plot the estimates we can use the `event_plot` (`ssc install event_plot, replace`) command where we restrict the figure to 10 leads and lags: 


```applescript
	event_plot e(b_iw)#e(V_iw), default_look graph_opt(xtitle("Periods since the event") ytitle("Average effect") xlabel(-10(1)10) ///
		title("eventstudyinteract")) stub_lag(L_#) stub_lead(F_#) trimlag(10) trimlead(10) together
```

And we get:

<img src="../../../assets/images/eventstudyinteract_1.png" height="300">

*INCOMPLETE*