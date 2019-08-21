# pg financial functions

plpgsql + plpython3 based financial functions

<!--
> select
>   not exists (select 1 from pg_language where lanname='plpython3u') as has_no_plpython3u
> \gset
> \if :has_no_plpython3u
>   create language plpython3u;
> \endif
-->

## install

needs,
[plpython3u](https://www.postgresql.org/docs/current/plpython.html),
[scipy](https://www.scipy.org/install.html)
see [dockerfile](dockerfile)

```
psql -f src\readme.sql
```
## finan schema

<!--
> drop schema if exists finan_tests cascade;
> create schema finan_tests;
-->

<!--
> drop schema if exists finan cascade;
> create schema finan;
-->

```
> set schema 'finan';
```

- [fixed-rate functions](#fixed-rate-functions)
- [discounted cash-flow analysis](#discounted-cash-flow-analysis)
- [black-scholes option pricing](#black-scholes-option-pricing)


## [fixed-rate functions](#fixed-rate-functions)

### `future_value(rate, nper, pmt, pv=0, due=0/1)`

<!--
> create or replace function future_value(
>     rate double precision,
>     nper double precision,
>     pmt double precision default 0,
>     pv double precision default 0,
>     due int default 0) -- end: 0, begin: 1
> returns double precision as $$
>     import numpy as np
>     return np.fv(rate, nper, pmt, pv, due)
> $$ language plpython3u immutable strict;

> create or replace function finan_tests.test_future_values() returns setof text as $$
> begin
>   return next ok(floor(future_value(0.1/4, 4*4, -2000, 0, 1)) = 39729, 'can calc');
>   return next ok(future_value(null, 4*4, -2000) is null, 'is strict');
> end;
> $$ language plpgsql;
-->

invest 1000/month for 5 years with compounded at 5%/yr
```
> select future_value(0.05/12, 5*12, -1000);
future_value
------------------
68006.0828408428
```

invest on start of quarters, 2000/q for 4 years with rate 10%/year
```
> select future_value(0.1/4, 4*4, -2000, 0, 1);
future_value
------------------
39729.4608941661
```

### `present_value(rate, nper, pmt, fv=0, due=0/1)`

<!--
> create or replace function present_value(
>   rate double precision,
>   nper double precision,
>   pmt double precision,
>   fv double precision default 0,
>   due int default 0) -- end: 0, begin: 1
> returns double precision as $$
>   import numpy as np
>   return np.pv(rate, nper, pmt, fv, due)
> $$ language plpython3u strict;

> create or replace function finan_tests.test_present_values() returns setof text as $$
> begin
>   return next ok(floor(present_value(0.045/12, 5*12, -93.22)) = 5000, 'can calc');
>   return next ok(present_value(null, 5*12, -93.22) is null, 'is strict');
> end;
> $$ language plpgsql;
-->

cd pays 100/mo with 5.5%/year for 5 years. buy if less than present-value
```
> select present_value(0.055/12, 5*12, 100);
present_value
------------------
-5235.2835445651
```

a loan of 4.5%, 93.22 payment for 5 years. the original loan is:
```
> select present_value(0.045/12, 5*12, -93.22);
present_value
------------------
5000.26303638651
```

### `payment(rate, nper, pv, fv=0, due=0/1)`

<!--
> create or replace function payment(
>   rate double precision,
>   nper double precision,
>   pv double precision,
>   fv double precision default 0,
>   due int default 0) -- end: 0, begin: 1
> returns double precision as $$
>   import numpy as np
>   return np.pmt(rate, nper, pv, fv, due)
> $$ language plpython3u immutable strict;

> create or replace function finan_tests.test_payment() returns setof text as $$
> begin
>   return next ok(trunc(payment(0.045/12, 5*12, 5000)) = -93, 'can calc');
>   return next ok(payment(0.045, 5*12, null) is null, 'is strict');
> end;
> $$ language plpgsql;
-->

a loan of 5000, with 4.5%, for 5 years. payment per period is:
```
> select payment(0.045/12, 5*12, 5000);
    payment
------------------
-93.215096207585
```


### `number_of_periods(rate, pmt, pv, fv=0, due=0/1)`

<!--
> create or replace function number_of_periods (
>   rate double precision,
>   pmt double precision,
>   pv double precision,
>   fv double precision default 0,
>   due int default 0) -- end: 0, begin: 1
> returns double precision as $$
>   import numpy as np
>   return np.nper(rate, pmt, pv, fv, due)
> $$ language plpython3u immutable strict;
-->

a loan of 5000, with 4.5%, paid 100/mo, will take
```
> select number_of_periods(0.045/12, -100, 5000);
    periods
------------------
55.4742521906629
```

### `rate(nper, pmt, pv, fv=0, due=0/1, guess=0.1, tol=1e-6, max_iter=1000)`
<!--
> create or replace function rate (
>   nper double precision,
>   pmt double precision,
>   pv double precision,
>   fv double precision default 0,
>   due int default 0, -- end: 0, begin: 1
>   guess double precision default 0.1,
>   tol double precision default 1e-6,
>   max_iter int default 1000)
> returns double precision as $$
>   import numpy as np
>   return np.rate(nper, pmt, pv, fv, due, guess, tol, max_iter)
> $$ language plpython3u immutable strict;
-->

a loan of 5000, paid 100/mo, for 5 years
```
> select rate(5 * 12, -93.22, 5000) * 12;
    ?column?
--------------------
0.0450215684902132
```

### `principal_payment(rate, per, nper, pv, fv=0, due=0/1)`
<!--
> create or replace function principal_payment(
>     rate double precision,
>     per double precision,
>     nper double precision,
>     pv double precision,
>     fv double precision default 0,
>     due int default 0) -- end: 0, begin: 1
> returns double precision as $$
>     import numpy as np
>     return np.ppmt(rate, per, nper, pv, fv, due)
> $$ language plpython3u immutable strict;
-->

### `interest_payment(rate, per, nper, pv, fv=0, due=0/1)`
<!--
> create or replace function interest_payment(
>   rate double precision,
>   per double precision,
>   nper double precision,
>   pv double precision,
>   fv double precision default 0,
>   due int default 0) -- end: 0, begin: 1
> returns double precision as $$
>   import numpy as np
>   return np.ipmt(rate, per, nper, pv, fv, due)
> $$ language plpython3u immutable strict;
-->

a loan of 5000, with 4.5%, for 5 years. payment per period is, on the 12th month,
```
> select payment(0.045/12, 5*12, 5000);
    payment
------------------
-93.215096207585
> select principal_payment(0.045/12, 12, 5*12, 5000);
principal_payment
-------------------
-77.595028342707
> select interest_payment(0.045/12, 12, 5*12, 5000);
interest_payment
------------------
-15.620067864878
```

## [discounted cash-flow analysis](#discounted-cash-flow-analysis)

for rate-of-return/net-present-value over an investment cashflow.
rate is the alternative interest rate from other source vs the investment cashflow

### `net_present_value(rate, cashflow=[])`

<!--
> create or replace function net_present_value(
>   rate double precision,
>   cashflow double precision[])
> returns double precision as $$
>   import numpy as np
>   return np.npv(rate, cashflow)
> $$ language plpython3u immutable strict;

> create or replace function net_present_value(
>   rate double precision[],
>   cashflow double precision[])
> returns double precision as $$
>   import numpy as np
>   return [np.npv(r, cashflow) for r in rate]
> $$ language plpython3u immutable strict;
-->

given a discount-rate 10%, a 10000 investments returns cashflow as below. this investment's worth now is:
```
> select net_present_value(
>   0.1,
>   array[-10000,3000,4200,6800]::double precision[]);
net_present_value
-------------------
1307.28775356874
```
excel has a [different](https://feasibility.pro/npv-calculation-in-excel-numbers-not-match/) way to calc


### `internal_rate_of_return(rate, cashflow=[])`
<!--
> create or replace function internal_rate_of_return(
>   cashflow double precision[])
> returns double precision as $$
>   import numpy as np
>   return np.irr(cashflow)
> $$ language plpython3u immutable strict;
-->

given a set of cashflow, what rate is which gives net-present-value = 0
```
> select internal_rate_of_return(
>   array[-10000,3000,4200,6800]::double precision[]);
internal_rate_of_return
-------------------------
    0.163405600688989
```

### `modified_internal_rate_of_return(cashflow=[], rate, reinvest_rate)`
<!--
> create or replace function modified_internal_rate_of_return(
>   cashflow double precision[],
>   rate double precision,     -- rate on cashflow
>   reinvest_rate double precision) -- rate on cashflow reinvestment
> returns double precision as $$
>   import numpy as np
>   return np.mirr(cashflow, rate, reinvest_rate)
> $$ language plpython3u immutable strict;

> create or replace function finan_tests.test_modified_internal_rate_of_return() returns setof text as $$
> begin
>   return next ok(trunc(modified_internal_rate_of_return(
>       array[-10000,3000,4200,6800]::double precision[], 0.1, 0.12) * 100) = 15, 'can calc');
>   return next ok(modified_internal_rate_of_return(null, null, null) is null, 'is strict');
>   return next ok((modified_internal_rate_of_return(
>       array[]::double precision[], 0.1, 0.12) = 'NaN'), 'nan on empty cashflow');
> end;
> $$ language plpgsql;

-->

a 10k loan generates a cashflow. a 10% interest for the 10k loan, and 12% for reinvested profits
```
 > select modified_internal_rate_of_return(
 >   array[-10000,3000,4200,6800]::double precision[], 0.1, 0.12);
modified_internal_rate_of_return
----------------------------------
                0.151471336646763
```


### `net_present_value(rate, cashflow=[], dates=[])`

<!--
ref: https://stackoverflow.com/questions/8919718/financial-python-library-that-has-xirr-and-xnpv-function

> create or replace function net_present_value(
>   rate double precision,
>   cashflow double precision[],
>   dates date[] )
> returns double precision as $$
>   import datetime
>   if rate <= -1.0:
>       return float('inf')
>   date_vals = list(map(lambda x: datetime.datetime.strptime(x,'%Y-%m-%d').date(), dates))
>   d0 = date_vals[0]
>   return sum([ vi / (1.0 + rate) ** ((di - d0).days / 365) for vi, di in zip(cashflow, date_vals)])
> $$ language plpython3u immutable strict;
-->

a -1000 loan at 1/1/2018, returns cashflow as below with 10% discount rate; its values now is:
```
> select net_present_value(
>   0.1,
>   array[-1000, 250, 250, 250, 250, 250]::double precision[],
>   array['2018-1-1', '2018-6-1', '2018-12-1', '2019-3-1', '2019-9-1', '2019-12-30']::date[]);
net_present_value
-------------------
113.271525238905
```

### `internal_rate_of_return(cashflow=[], dates=[])`

<!--
> create or replace function internal_rate_of_return(
>   cashflow double precision[],
>   dates date[] )
> returns double precision as $$
>   import scipy.optimize
>   import datetime
>   date_vals = list(map(lambda x: datetime.datetime.strptime(x,'%Y-%m-%d').date(), dates))

>   def xnpv (rate, casflow, dates):
>       if rate <= -1.0:
>           return float('inf')
>       d0 = date_vals[0]
>       return sum([ vi / (1.0 + rate) ** ((di - d0).days / 365) for vi, di in zip(cashflow, dates)])

>   try:
>       return scipy.optimize.newton(lambda r: xnpv(r, cashflow, date_vals), 0.0)
>   except RuntimeError:    # Failed to converge?
>       return scipy.optimize.brentq(lambda r: xnpv(r, cashflow, date_vals), -1.0, 1e10)

> $$ language plpython3u immutable strict;
-->

a -1000 loan at 1/1/2018, returns cashflow as below; rate of return for net-present-value ~ 0
```
> select internal_rate_of_return(
>   array[-1000, 250, 250, 250, 250, 250]::double precision[],
>   array['2018-1-1', '2018-6-1', '2018-12-1', '2019-3-1', '2019-9-1', '2019-12-30']::date[]);
internal_rate_of_return
-------------------------
    0.204099471443879

> select net_present_value(
>   0.204099471443879,
>   array[-1000, 250, 250, 250, 250, 250]::double precision[],
>   array['2018-1-1', '2018-6-1', '2018-12-1', '2019-3-1', '2019-9-1', '2019-12-30']::date[]);
net_present_value
----------------------
3.38218342221808e-12
```

## [black-scholes option pricing](#black-scholes-option-pricing)

for black-scholes model options pricing and it sensitivity measures

<!--
> create or replace function normpdf(
>   x double precision,
>   loc double precision default 0.0,
>   scale double precision default 1.0
> ) returns double precision as $$
>   import scipy.stats
>   return scipy.stats.norm.pdf(x,loc,scale)
> $$ language plpython3u immutable strict;


> create or replace function normcdf(
>   x double precision,
>   loc double precision default 0.0,
>   scale double precision default 1.0
> ) returns double precision as $$
>   import scipy.stats
>   return scipy.stats.norm.cdf(x,loc,scale)
> $$ language plpython3u immutable strict;
-->

Call - option to buy on or before date at certain price,
Put  - option to sell on or before date at certain price;
like an insurance for a trade. where:

```
> create type black_scholes_t as (
>   SO double precision, -- current price
>   X double precision, -- exercise price of option
>   r double precision, -- risk-free rate over option period
>   T double precision, -- option expiration (in years)
>   S double precision, -- asset volatility
>   q double precision -- asset yield
> );
```
<!--
> create or replace function d1(
>   a black_scholes_t
> ) returns double precision as $$
>   select (ln(a.SO / a.X) + (a.r + a.S*a.S/2.0) * a.T) / (a.S * sqrt(a.T))
> $$ language sql immutable strict;
-->

### `[callPrice, putPrice] = price(black_scholes_t)`
for call and put option pricing

<!--
> create or replace function price(
>   a black_scholes_t
> ) returns double precision[2] as $$
> declare
>   d1 double precision = d1(a);
>   d2 double precision = d1 - (a.S * sqrt(a.T));
>   ert double precision = exp(-a.r * a.T);
>   eqt double precision = exp(-a.q * a.T);
> begin
>   return array[
>       a.SO * eqt * normcdf(d1) - a.X * ert * normcdf(d2), -- call
>       a.X * ert * normcdf(-d2) - a.SO * eqt * normcdf(-d1) -- put
>   ]::double precision[2];
> end;
> $$ language plpgsql immutable strict;
-->
an option expired in 3 months at an exercise price of $95. Trading at at $100, and volatility of 50% per annum, with risk-free rate of 10%.
```
> select ((100, 95, 0.1, 0.25, 0.5, 0.0)::black_scholes_t).price;
                price
-------------------------------------
{13.6952727386081,6.34971438129973}
```

s&p-100 is at 910 with 25% volatility and 2.5% dividend, risk-free rate is 2%. what is strike price at 980 for 3 months options
```
> select ((910, 980, 0.02, 0.25, 0.25, 0.025)::black_scholes_t).price;
                price
-------------------------------------
{19.6366634066511,90.4186565481906}
```

option to buy GBP with USD in 4 months at 1.6, when 8% USD interest and 11% GBP and USD volatility at 20%
```
> select ((1.6, 1.6, 0.08, 0.3333, 0.2, 0.11)::black_scholes_t).price;
                price
-----------------------------------------
{0.0603272194547098,0.0758270545223216}
```


### `[callDelta, putDelta] = delta(black_scholes_t)`
for sensitivity to price change

<!--
> create or replace function delta(
>   a black_scholes_t
> ) returns double precision[2] as $$
> declare
>   d1 double precision = d1(a);
>   eqt double precision = exp(-a.q * a.T);
>   cd double precision = eqt * normcdf(d1);
> begin
>   return array[
>       cd,
>       cd - eqt
>   ]::double precision[2];
> end;
> $$ language plpgsql immutable strict;
-->

```
> select ((50, 50, 0.1, 0.25, 0.3, 0)::black_scholes_t).delta;
                delta
----------------------------------------
{0.595480769902361,-0.404519230097639}
```
### `gamma(black_scholes_t)`
for sensitivity to delta change

<!--
> create or replace function gamma(
>   a black_scholes_t
> ) returns double precision as $$
> declare
>   d1 double precision = d1(a);
> begin
>   return (normpdf(d1) * exp(-a.q * a.t)) / (a.SO * a.s * sqrt(a.T));
> end;
> $$ language plpgsql immutable strict;
-->
```
> select ((50, 50, 0.1, 0.25, 0.3, 0)::black_scholes_t).gamma;
    gamma
--------------------
0.0516614748457897
```

### `[CallEl, PutEl] = labmda(black_scholes_t)`
for option elasticity - % change in option price over change in asset-price (SO)

<!--
> create or replace function lambda(
>   a black_scholes_t
> ) returns double precision[2] as $$
> declare
>   d1 double precision = d1(a);
>   nd1 double precision = normcdf(d1);

>   px double precision[2] = price(a);
>   cp double precision = px[1];
>   pp double precision = px[2];

>   ce double precision;
>   pe double precision;
> begin
>   if cp>=(1e-14) and pp>=(1e-14) then
>       ce = a.SO / cp * nd1;
>       if nd1-1 < 1e-6 then
>           pe = a.SO / pp * (-normcdf(-d1));
>       else
>           pe = a.SO / pp * (nd1 - 1);
>       end if;
>   end if;
>   return array[ce, pe];
> end;
> $$ language plpgsql immutable strict;
-->

```
> select ((50, 50, 0.12, 0.25, 0.3, 0)::black_scholes_t).lambda;
                lambda
--------------------------------------
{8.12738492654569,-8.64655992790169}
```

### `[CallRho, PutRho] = rho(black_scholes_t)`
% change in option price over change in interest rate (r)

<!--
> create or replace function rho(
>   a black_scholes_t
> ) returns double precision[2] as $$
> declare
>   d1 double precision = d1(a);
>   d2 double precision = d1 - (a.S * sqrt(a.T));
>   d3 double precision = a.X * a.T * exp(-a.r * a.T);
> begin
>   return array[d3 * normcdf(d2), -d3 * normcdf(-d2)];
> end;
> $$ language plpgsql immutable strict;
-->

```
> select ((50, 50, 0.12, 0.25, 0.3, 0)::black_scholes_t).rho;
                rho
--------------------------------------
{6.66863756134086,-5.46193160801549}
```

### `[CallTheta, PutTheta] = theta(black_scholes_t)`
% change in option price over change in time (T)

<!--
> create or replace function theta(
>   a black_scholes_t
> ) returns double precision[2] as $$
> declare
>   d1 double precision = d1(a);
>   d2 double precision = d1 - (a.S * sqrt(a.T));

>   b double precision = -a.SO * normpdf(d1) * a.S * exp(-a.q * a.T) / (2.0 * sqrt(a.T));
>   eqt double precision = a.q * a.SO * exp(-a.q * a.T);
>   ert double precision = a.r * a.X * exp(-a.r * a.T);
> begin
>   return array[
>       b + normcdf(D1) * eqt - ert * normcdf(D2),
>       b - normcdf(-D1) * eqt + ert * normcdf(-D2)
>   ];
> end;
> $$ language plpgsql immutable strict;
-->

```
> select ((50, 50, 0.12, 0.25, 0.3, 0)::black_scholes_t).theta;
                theta
---------------------------------------
{-8.96302975902918,-3.14035655773813}
```

### `vega(black_scholes_t)`
% change in option price over volatility (S)

<!--
> create or replace function vega(
>   a black_scholes_t
> ) returns double precision as $$
>   select a.SO * sqrt(a.T) * normpdf(d1(a)) * exp(-a.q * a.T)
> $$ language sql immutable strict;
-->

```
> select ((50, 50, 0.12, 0.25, 0.3, 0)::black_scholes_t).vega;
    vega
------------------
9.60347288264262
```

<!--
> set search_path to default;
-->

<!--
> select exists (select 1 from pg_available_extensions where name='pgtap') as has_pgtap
> \gset

> \if :has_pgtap
> set search_path TO finan_tests, finan, public;
> create extension if not exists pgtap with schema finan_tests;
> select * from runtests('finan_tests'::name);
> \endif

> drop schema finan_tests cascade;
-->
