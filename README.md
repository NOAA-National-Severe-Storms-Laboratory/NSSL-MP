# NSSL-MP
NSSL Cloud Microphysics scheme (MPAS/CM1 etc.)

The main routine (module_mp_nssl_2mom.F90) is common to MPAS and CCPP, but 
separate drivers (mp_nssl.F90) are located in the MPAS and CCPP subdirectories.
The Makefile is used for MPAS, and mp_nssl.meta is for CCPP


Some background information and usage tips for the NSSL microphysics scheme.

## NOTES ON ADVECTION: 

### MPAS: 
 The advection scheme in MPAS can result in noisy values at the edges of reflectivity cores. This is because the errors in the moments for number and mass are mismatched and can end up with small amounts of large hydrometeors. Some reduction can be achieved by setting config_coef_3rd_order to a value closer to 1 (e.g., 0.9 vs. default value of 0.25) to reduce the 4th-order component. 
### WRF:  
   Best results are attained using WENO (Weighted Essentially Non-Oscillatory) scalar advection option. This helps to limit oscillations at the edges of precipitation regions (i.e., sharp gradient), which in turns helps to prevent mismatches of moments that can show up as noisy reflectivity values.
```
   moist_adv_opt  = 4,
   scalar_adv_opt = 3,
```
 The monotonic option (2) is less effective, but better than the default positive definite option (1)


# DESCRIPTION:

The NSSL bulk microphysical parameterization scheme describes form and phase changes among a range of liquid and ice hydrometeors, as described in Mansell et al. (2010) and Mansell and Ziegler (2013). It is designed with deep (severe) convection in mind at grid spacings of up to 4 km, but can also be run at larger grid spacing as needed for nesting etc. It is also able to capture non-severe and winter weather. The scheme predicts the mass mixing ratio and number concentration of cloud droplets, raindrops, cloud ice crystals (columns), snow particles (including large crystals and aggregates), graupel, and (optionally) hail. The 3-moment option additionally predicts the 6th moments of rain, graupel, and hail which in turn predicts the PSD shape parameters (set config_nssl_3moment=.true.). The hail variables can be turned off via the config_nssl_ccn_on flag.

Although the scheme can be run for large scales, it is more suited for dx <= 4km (e.g., regional MPAS). The scheme uses the dt_microp parameter for sub-stepping the microphysics to maintain stability for large time steps (for dt > 75s). This has not been thoroughly tested in MPAS but is stable in FV3 regression tests. It is not otherwise 'scale-aware' currently.

The default NSSL scheme (2-moment) has the separate hail species turned with predicted bulk density for graupel and hail, and predicted CCN concentration

# Usage

To select NSSL in the physics namelist:

### MPAS:
```
  config_microp_scheme = 'mp_nssl2m' 
```
### WRF: 
```
  mp_physics = 18 
```
  The legacy options (17,19,21,22, see below) still behave as before (for now), but going
  forward one should use mp_physics=18 with modifier flags. 2025 Update, however,
  sets nssl_ccn_on=1 by default (keeps supersaturation much more reasonable; except for single moment).

### UFS-FV3: 
```
  imp_physics = 17
  nwat = 7 ! 7 with hail, but 6 if hail is deactivated
```

## Option flags/parameters:

### MPAS:
```
 config_nssl_3moment : (logical) default value of .false., setting to .true. adds 6th moment for rain, graupel (i.e., 3-moment ) and hail (Only needed for turning  3-moment on)
 config_nssl_ccn_on : (logical) predicted CCN concentration: default is on (.true.) 
 config_nssl_hail_on : (logical) If not set explicitly, it is set automatically to true. Set to false to run with graupel only (non-severe deep convection)
```

### UFS-FV3:
```
 nssl_3moment : (logical) default value of .false., setting to .true. adds 6th moment for rain, graupel (i.e., 3-moment ) and hail (Only needed for turning  3-moment on)
 nssl_ccn_on : (logical) predicted CCN concentration: default is on (.true.) 
 nssl_hail_on : (logical) If not set explicitly, it is set automatically to true. Set to false to run with graupel only (non-severe deep convection)
```

### WRF:
```
 nssl_3moment : (integer) default value of 0, setting to 1 adds 6th moment for rain, graupel (i.e., 3-moment ) and hail (Only needed for turning  3-moment on)
 nssl_ccn_on : (integer) predicted CCN concentration: default is on (=1) 
 nssl_hail_on : (integer) If not set explicitly, it is set automatically to 1. Set to 0 to run with graupel only (non-severe deep convection)
 nssl_density_on : (integer) If not set explicitly, it is set automatically to 1. Set to 0 to turn off graupel/hail density prediction and use fixed values
```

  Note: Graupel/hail density prediction is currently always turned on, and the CCN category is always treated as the number of *activated* CCN.

### Other namelist options (MPAS, WRF/UFS-FV3) and default values (also "physics" namelist)
   
```
   config_nssl_alphar/ nssl_alphar  = 0.    ! (real) PSD shape parameter for rain (2-moment) (not included in WRF as of 4.7.1 but available in nssl_mp_params namelist)
   config_nssl_alphah/ nssl_alphah  = 0.    ! (real) PSD shape parameter for graupel (2-moment)
   config_nssl_alphahl / nssl_alphahl = 1.    ! (real) PSD shape parameter for hail (2-moment)
   config_nssl_ehw0 / nssl_ehw0    = 0.9   ! (real) Maximum graupel-droplet collection efficiency
   config_nssl_ehlw0 / nssl_ehlw0   = 0.9   ! (real) Maximum hail-droplet collection efficiency
```

```
   config_nssl_cccn / nssl_cccn  - (real) Initial background concentration of cloud condensation 
                       nuclei (per m^3 at sea level)
                   0.25e+9 maritime
                   0.5e+9 "low-med" continental
                   0.8e+9 "low-med" continental (DEFAULT)
                   1.0e+9 "med-high" continental
                   1.5e+09 - high-extreme continental CCN)
                   Larger values run a risk of unrealistically weak 
                   precipitation production
                 Value sets the concentration at MSL, and an initially
                 homogeneous number mixing ratio (nssl_ccn/1.225) is assumed throughout
                 the depth of the domain. The droplet concentration near cloud base
                 will be less than nssl_cccn because of the well-mixed assumption, 
                 so if a target Nc is desired, set nssl_cccn higher by a factor of 
                 1.225/(air density at cloud base).
```

## More background

The graupel and hail particle densities are also calculated by predicting the total particle volume. The graupel category therefore emulates a range of characteristics from high-density frozen drops (includes small hail) to low-density graupel (from rimed ice crystals/snow) in its size and density spectrum. The hail category is designed to simulate larger hail sizes. Hail is only produced from higher-density large graupel that is actively riming (esp. in wet growth).

Hydrometeor size distributions are assumed to follow a gamma functional form. Microphysical processes include cloud droplet and cloud ice nucleation, condensation, deposition, evaporation, sublimation, collection–coalescence, variable-density riming, shedding, ice multiplication, cloud ice aggregation, freezing and melting, and conversions between hydrometeor categories. 

Cloud concentration nuclei (CCN) concentration is predicted as in Mansell et al. (2010)  with a bulk activation spectrum approximating small aerosols. (New option nssl_ccn_is_ccna=1 instead predicts the number of activated CCN.) The model tracks the number of unactivated CCN, and the local CCN concentration is depleted as droplets are activated, either at cloud base or in cloud. The CCN are subjected to advection and subgrid turbulent mixing but have no other interactions with hydrometeors; for example, scavenging by raindrops is omitted. CCN are restored by droplet evaporation and by a gradual regeneration when no hydrometeors are present (ccntimeconst). Aerosol sensitivity is enhanced by explicitly treating droplet condensation instead of using a saturation adjustment. Supersaturation (within reason) is allowed to persist in updraft with low droplet concentration.

Droplet activation option method is controlled by the 'irenuc' option (internal to NSSL module). Default (old) option (2) depletes CCN from unactivated CCN field. New option (5) instead counts the number of activated CCN (nucleated droplets) with the assumption of an initial constant CCN number mixing ratio. Option 5 better handles supersaturation at low droplet number concentration (e.g., maritime or high scavenging rates) concentrations by allowing extra droplet activation at high SS.

```
  irenuc     : (nssl_mp_params namelist)
               2 = ccn field is UNactivated aerosol (default; old droplet activation)
                   Can switch to counting activated CCN with nssl_ccn_is_ccna=1
               5 = ccn field must be ACTVIATED aerosol (new default as of Feb. 2025)
                   Must have nssl_ccn_on=1 for irenuc=5
                   Allows activation beyond limit of nssl_cccn at higher supersaturation
                   as an approximation of nucleation mode aerosol being activated. (Mainly
                   an issue for low CCN concentration with deep updrafts.)
                   If more strict limitation of activation is desired, use option 7.
               7 = ccn field must be ACTVIATED aerosol (new droplet activation) 
                   Must have nssl_ccn_on=1 for irenuc=7
```

Excessive size sorting (common in 2-moment schemes) is effectively controlled by an adaptive breakup method that prevents reflectivity growth by sedimentation (Mansell 2010). For 2-moment, infall=3 (default; nssl_mp_params namelist) is recommended. For 3-moment, infall only applies to droplets, cloud ice, and snow, since no corrections are needed for the 3-moment species (rain, graupel, hail).

Graupel -> hail conversion: The parameter ihlcnh selects the method of converting graupel (hail embryos) to the hail category. The default value is -1 for automatic setting. The original option (ihlcnh=1) is replaced by a new option (ihlcnh=3) as of May 2023. ihlcnh=3 converts from the graupel spectrum itself based on the wet growth diameter, which generally results in fewer initiated hailstones with larger diameters (and larger mean diameter at the ground).

June 2023 (WRF 4.5.x) update introduces changes in the default options for graupel/hail fall speeds and collection efficiencies. The original fall speed options (icdx=3; icdxhl=3) from Mansell et al. (2010) are switched to the Milbrandt and Morrison (2013) fall speed curves (icdx=6; icdxhl=6). Because the fall speeds are generally a bit lower, a partially compensating increase in maximum collection efficiency is set by default: ehw0/ehlw0 increased to 0.9. One effect is somewhat reduced total precipitation and cold pool intensity for supercell storms.

```
  (nssl_mp_params namelist)
  icdx         - fall speed option for graupel (was 3, now is 6)
  icdxhl       - fall speed option for hail (was 3, now is 6)
  ehw0,ehlw0   - Maximim droplet collection efficiencies for graupel (ehw0=0.75, now 0.9)
                 and hail (ehlw0=0.75, now 0.9) 
```

In summary, to get something closer to previous behavior, use the following:

```
&nssl_mp_params
  icdx   = 3
  icdxhl = 3
  ehw0   = 0.5
  ehlw0  = 0.75
  ihlcnh = 1
/
```

## Snow Aggregation and reflectivity:

Snow self-collection (aggregation) has been curbed in the 4.5.x version by reducing the collision efficiency and the temperature range over which aggregation is allowed (esstem):

```
  ess0 = 0.5 ! collision efficiency, reduced from 1 to 0.5
  esstem1 = -15. ! was -25.  ! lower temperature where snow aggregation turns on
  esstem2 = -10. ! was -20.  ! higher temperature for linear ramp of ess from zero at esstem1 to formula value at esstem2
```

  If desired, some further reduction in aggregation can be gained from setting iessopt=4, which reduces ess0 to 0.1 (80% reduction) in conditions of ice subsaturation (RHice < 100%).
  Snow reflectivity formerly had a default setting that turned on a crude bright band enhancement (iusewetsnow=1). This is now turned off by default (iusewetsnow=0)
  These snow parameters can be accessed through the nssl_mp_params namelist.

## WRF Legacy (obsolete) mp_physics options:

```
 mp_physics 
  = 22 ! NSSL scheme (2-moment) without hail
      Equivalent: mp=18, nssl_hail_on=0, nssl_ccn_on=1
  = 17 ! NSSL scheme (2-moment) with hail is now the same as mp=18
      Equivalent: mp=18, nssl_ccn_on=1 <- must explicitly set nssl_ccn_on=0 to 
      get old behavior
  = 19, NSSL 1-moment (7 class: qv,qc,qr,qi,qs,qg,qh; predicts graupel density)
      Equivalent: mp=18, nssl_2moment_on=0, nssl_ccn_on=1 (do no set nssl_hail_on)
  = 21, NSSL 1-moment, (6-class), very similar to Gilmore et al. 2004
      Equivalent: mp=18, nssl_2moment_on=0, nssl_hail_on=0, nssl_ccn_on=0, 
      nssl_density_on=0
```

# References:

Mansell, E. R., C. L. Ziegler, and E. C. Bruning, 2010: Simulated electrification 
   of a small thunderstorm with two-moment bulk microphysics. J. Atmos. Sci., 
   67, 171-194, doi:10. 1175/2009JAS2965.1.

Mansell, E. R. and C. L. Ziegler, 2013: Aerosol effects on simulated storm
  electrification and precipitation in a two-moment bulk microphysics model. 
  J. Atmos. Sci., 70 (7), 2032-2050, doi:10.1175/JAS-D-12-0264.1.

Mansell, E. R., D. T. Dawson, J. M. Straka, Bin-emulating Hail Melting in 3-moment 
   bulk microphysics, J. Atmos. Sci., 77, 3361-3385, doi: 10.1175/JAS-D-19-0268.1

Ziegler, C. L., 1985: Retrieval of thermal and microphysical variables in observed
    convective storms. Part I: Model development and preliminary testing. J. 
    Atmos. Sci., 42, 1487-1509.

## Sedimentation reference:

Mansell, E. R., 2010: On sedimentation and advection in multimoment bulk microphysics.
    J. Atmos. Sci., 67, 3084-3094, doi:10.1175/2010JAS3341.1.




