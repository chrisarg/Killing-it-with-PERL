

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

|Function|Mean Difference    |
|:------:|:----------------:|
|Sqrt| 0.00e+00|
|Sin| 9.98e-21|
|Cos| -6.07e-21|
|Exp| 6.69e-19|
|Log| 3.54e-19|

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


