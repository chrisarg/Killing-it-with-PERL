---
title: " The Quest for Performance Part IV : May the SIMD Force be with you "
date: 2024-07-09
---

|Function|Library|Execution Time    |
|:------:|:-----:|:----------------:|
|   Sqrt | numpy |  1.02e-01 seconds|
|    Log | numpy |  2.82e-01 seconds|
|    Exp | numpy |  3.00e-01 seconds|
|    Cos | numpy |  3.55e-01 seconds|
|    Sin | numpy |  4.83e-01 seconds|
|    Sin | numba |  1.05e-01 seconds|
|   Sqrt | numba |  1.05e-01 seconds|
|    Exp | numba |  1.27e-01 seconds|
|    Log | numba |  1.47e-01 seconds|
|    Cos | numba |  1.82e-01 seconds|

|Function|Mean ULP   | Median ULP | 99.9th ULP |Max ULP |
|:------:|:----------------:|:----------------:|:----------------:|:----------------:|
|Sqrt| 0.00e+00|0.00e+00|0.00e+00|0.00e+00
|Sin| 1.56e-03|0.00e+00|1.00e+00|1.00e+00
|Cos| 1.43e-03|0.00e+00|1.00e+00|1.00e+00
|Exp| 5.47e-03|0.00e+00|1.00e+00|2.00e+00
|Log| 1.09e-02|0.00e+00|2.00e+00|3.00e+00

|Function|Library|Execution Time    |
|:------:|:-----:|:----------------:|
|     Sqrt | PDL | 1.11e-01 seconds|
|     Log  | PDL | 2.73e-01 seconds|
|     Exp  | PDL | 3.10e-01 seconds|
|     Cos  | PDL | 3.54e-01 seconds|
|     Sin  | PDL | 4.75e-01 seconds|
|Sqrt | C | 1.23e-01 seconds|
|Log | C | 2.87e-01 seconds|
|Exp | C | 3.19e-01 seconds|
|Cos | C | 3.57e-01 seconds|
|Sin | C | 4.96e-01 seconds|

|Function|Library|Execution Time    |
|:------:|:-----:|:----------------:|
|Sqrt | C - Ofast| 1.00e-01 seconds|
|Sin | C - Ofast| 9.89e-02 seconds|
|Cos | C - Ofast| 1.05e-01 seconds|
|Exp | C - Ofast| 8.40e-02 seconds|
|Log | C - Ofast| 1.04e-01 seconds|


