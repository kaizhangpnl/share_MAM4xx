## MAM4xx-ZM coupling 

## Convective precipitation and mass fluxes 

ZM exports the expected fields used by ConvProc:

- Updraft/downdraft mass fluxes: `zm_mflx_up`, `zm_mflx_dn` (kg m⁻² s⁻¹)
- Entrainment/detrainment: `zm_entr_up`, `zm_detr_up`, `zm_entr_dn` (s⁻¹)
- Rain production: `zm_rain_prod` (kg kg⁻¹ s⁻¹)
- In-cloud liquid: `zm_icw_liq` / `zm_ql` (kg kg⁻¹)
- Cloud levels: `zm_cld_top` / `zm_jctop`, `zm_cld_bot` / `zm_jcbot`
- Other: `zm_mass_flux_int`, `zm_detr_qc`, `evap_dq_out`, etc.

Units handling of dry pressure thickness (`pseudo_density_dry` in Pa) is converted to millibars for ConvProc (`pa_to_mb = 0.01`). This addresses an earlier Copilot comment. ConvProc internally works in mb for pressure thickness and mb s⁻¹ for mass fluxes (converted from the kg m⁻² s⁻¹ inputs).

Precipitation production and evaporation rates are passed correctly as kg kg⁻¹ s⁻¹. Cloud fraction is estimated empirically from integrated mass flux (`dp1 = 0.018`, `dp2 = 500.0`, capped at 0.60).

**Open issue:** A known bug in `convproc.hpp` (and the original Fortran) uses a single scalar layer thickness `dz` (taken from the lowest level) for the Lagrangian updraft time step `dt_u = min(dz / wup, dt)` instead of level-varying thickness. This affects residence time, activation, and wet-removal rates. Reviewers (whannah1, DJFJJA) confirmed it; it is not fixed in this PR.

## Wet removal (in-cloud, below-cloud, resuspension)

The algorithm follows the established MAM4 / Wang et al. style:

- **In-cloud / updraft:** First-order loss using rain production rate `rprd` relative to in-cloud water and residence time derived from mass flux. Activation uses Abdul-Razzak & Ghan (modal).
- **Below-cloud:** Scavenging by precipitation flux; resuspension from evaporating precipitation via `faer_resusp_vs_fprec_evap_mpln` (log-normal raindrop polynomial).
- **Resuspension mapping:** `mmtoo_prevap_resusp` and `species_class` control which species are resuspended vs. remain scavenged (CAM5-style approach that tries to minimize impact on stratiform clouds).

These are wired through `mam4::wetdep::aero_model_wetdep` when `do_convproc` is true. Outputs include `aerdepwetis_convproc` (kg m⁻² s⁻¹).

## Convective tracer transport

ConvProc performs its own transport, activation, and wet removal using the ZM mass fluxes, entrainment/detrainment, and cloud levels. ZM itself can also transport tracers.

The author explicitly notes that consolidating convective vertical transport into one place is desirable but deferred to a follow-up PR (they do **not** want to remove ZM’s capability).

## Wet-Scavenging Process — Selected Output / Diagnostic Fields  

| Variable                  | Meaning                                              | Units          | Type / Layout | Notes |
|---------------------------|------------------------------------------------------|----------------|---------------|-------|
| `aerdepwetis_convproc`    | Convective interstitial aerosol wet deposition      | kg m⁻² s⁻¹     | 2D (per constituent) | New convective contribution |
| `aerdepwetis`             | Total interstitial aerosol wet deposition           | kg m⁻² s⁻¹     | 2D (per constituent) | Includes convective + stratiform |
| `aerdepwetcw`             | Cloud-borne aerosol wet deposition                  | kg m⁻² s⁻¹     | 2D (per constituent) | Cloud-borne fraction |
| `wetdep_hydrophilic_bc`   | Wet deposition of hydrophilic black carbon          | kg m⁻² s⁻¹     | 3D mid        | Species-specific |
| `drydep_hydrophilic_bc`   | Dry deposition of hydrophilic black carbon          | kg m⁻² s⁻¹     | 3D mid        | Species-specific |
| `wetdep_hydrophilic_oc`   | Wet deposition of hydrophilic organic carbon        | kg m⁻² s⁻¹     | 3D mid        | Species-specific |
| `drydep_hydrophilic_oc`   | Dry deposition of hydrophilic organic carbon        | kg m⁻² s⁻¹     | 3D mid        | Species-specific |
| `wetdep_dust_bin1`        | Wet deposition of dust bin 1                        | kg m⁻² s⁻¹     | 3D mid        | Dust bin |
| `wetdep_dust_bin2`        | Wet deposition of dust bin 2                        | kg m⁻² s⁻¹     | 3D mid        | Dust bin |
| `wetdep_dust_bin3`        | Wet deposition of dust bin 3                        | kg m⁻² s⁻¹     | 3D mid        | Dust bin |
| `wetdep_dust_bin4`        | Wet deposition of dust bin 4                        | kg m⁻² s⁻¹     | 3D mid        | Dust bin |
| `fracis`                  | Interstitial fraction of aerosol                    | none           | 3D mid        | Diagnostic |
| `dgnum`                   | Geometric mean diameter (dry)                       | m              | 3D mid (modes)| Diagnostic |
| `wetdens`                 | Wet density of aerosol                              | kg m⁻³         | 3D mid (modes)| Diagnostic |
| `qaerwat`                 | Aerosol water content                               | kg kg⁻¹        | 3D mid (modes)| Diagnostic |
| `dgnumwet`                | Geometric mean diameter (wet)                       | m              | 3D mid (modes)| Diagnostic |

**Notes**
- `aerdepwetis_convproc`, `aerdepwetis`, and `aerdepwetcw` are per-constituent (pcnst) surface fluxes.
- The `wetdep_*` / `drydep_*` fields are the usual species-specific deposition diagnostics (BC, OC, dust bins, and similarly for other species such as sulfate, sea salt, etc.).
- Several of these fields are also required inputs or intermediate diagnostics when `do_convproc` is true.

## Namelist Options for `mam4_wetscav` Convective Processing 

| Namelist Option              | Type    | Default | Description                                      | Notes |
|------------------------------|---------|---------|--------------------------------------------------|-------|
| `mam4_wetscav::convproc_do_aer` | logical | `false` | Enable convective processing of aerosols        | Used by ConvProc / wet scavenging with ZM outputs |
| `mam4_wetscav::convproc_do_gas` | logical | `false` | Enable convective processing of gases           | Currently often left false |
| `do_convproc` (internal)     | logical | —       | OR of the two flags above                       | Controls whether ConvProc path is activated |

**Notes**
- Both options are defined under the `mam4_wetscav` process (which inherits from `mam4_atm_proc_base`).
- In the PR testmods / shell commands, `convproc_do_aer=true` and `convproc_do_gas=false` are set.


## ZM Process Interface — Selected Convection-Related Fields 

| Variable | Meaning | Units | Type / Layout |
|----------|---------|-------|---------------|
| `zm_dcape` | Deep convective available potential energy tendency / rate of change | J kg⁻¹ s⁻¹ | 2D scalar |
| `zm_cape` | Convective available potential energy | J kg⁻¹ | 2D scalar |
| `zm_activity` | Indicator of convective activity | none | 2D scalar |
| `zm_jt` | Convective cloud-top level index (legacy name) | none (index) | 2D scalar |
| `zm_jctop` / `zm_cld_top` | Convective cloud-top level index | none (index) | 2D scalar |
| `zm_jcbot` / `zm_cld_bot` | Convective cloud-base / bottom level index | none (index) | 2D scalar |
| `zm_depth` | Convective layer depth | Pa | 2D scalar |
| `zm_mflx_up` | Updraft mass flux (mid-levels) | kg m⁻² s⁻¹ | 3D mid |
| `zm_mflx_dn` | Downdraft mass flux (mid-levels) | kg m⁻² s⁻¹ | 3D mid |
| `zm_mass_flux_int` | Mass flux at layer interfaces | kg m⁻² s⁻¹ | 3D interface |
| `zm_entr_up` | Updraft entrainment rate | s⁻¹ | 3D mid |
| `zm_detr_up` | Updraft detrainment rate | s⁻¹ | 3D mid |
| `zm_entr_dn` | Downdraft entrainment rate | s⁻¹ | 3D mid |
| `zm_rain_prod` | Convective rain production rate | kg kg⁻¹ s⁻¹ | 3D mid |
| `zm_icw_liq` / `zm_ql` | In-cloud liquid water mixing ratio | kg kg⁻¹ | 3D mid |
| `zm_detr_qc` | Detrainment tendency of cloud liquid mass | kg kg⁻¹ s⁻¹ | 3D mid |
| `zm_detr_qi` | Detrainment tendency of cloud ice mass | kg kg⁻¹ s⁻¹ | 3D mid |
| `zm_detr_nc` | Detrainment tendency of cloud liquid number | kg⁻¹ s⁻¹ | 3D mid |
| `zm_detr_ni` | Detrainment tendency of cloud ice number | kg⁻¹ s⁻¹ | 3D mid |
| `zm_prec` | Convective liquid precipitation rate at surface | m s⁻¹ | 2D scalar |
| `zm_snow` | Convective snow precipitation rate at surface | m s⁻¹ | 2D scalar |
| `evap_dq_out` | Evaporation tendency of water vapor | kg kg⁻¹ s⁻¹ | 3D mid |
| `evap_ds_out` | Evaporation tendency of dry static energy / temperature | K s⁻¹ | 3D mid |
| `deep_conv_t_tend` | Temperature tendency due to deep convection | K s⁻¹ | 3D mid |
| `mcsp_ds_out` | MCSP dry static energy / temperature tendency | K s⁻¹ | 3D mid |
| `mcsp_dq_out` | MCSP water-vapor tendency | kg kg⁻¹ s⁻¹ | 3D mid |
| `mcsp_du_out` | MCSP zonal-wind tendency | m s⁻² | 3D mid |
| `mcsp_dv_out` | MCSP meridional-wind tendency | m s⁻² | 3D mid |

**Notes**
- “Type / Layout” follows EAMxx conventions: 2D = column scalar, 3D mid = mid-level, 3D interface = interface.
- `zm_jctop`/`zm_cld_top` and `zm_jcbot`/`zm_cld_bot` are equivalent (renamed for clarity).
- `zm_icw_liq` and `zm_ql` refer to the same quantity (in-cloud liquid).
- MCSP fields are present when the Multiscale Coherent Structure Parameterization is enabled inside ZM.

- 
