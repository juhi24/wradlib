---
jupyter:
  jupytext:
    text_representation:
      extension: .md
      format_name: myst
      format_version: '1.3'
      jupytext_version: 1.17.3
---

```{raw-cell}
:tags: [hide-cell]
This notebook is part of the wradlib documentation: https://docs.wradlib.org.

Copyright (c) wradlib developers.
Distributed under the MIT License. See LICENSE.txt for more info.
```

# Hydrometeorclassification

```{code-cell} python
import wradlib as wrl
import wradlib_data
import xarray as xr
import os
import numpy as np
import matplotlib as mpl
import matplotlib.pyplot as plt
import warnings

warnings.filterwarnings("ignore")
try:
    get_ipython().run_line_magic("matplotlib inline")
except:
    plt.ion()
from scipy import interpolate
import datetime as dt
import glob
```

The hydrometeorclassification code is based on the paper by [Zrnic et.al 2001](https://dx.doi.org/10.1175/1520-0426%282001%29018%3C0892:TAPFAC%3E2.0.CO;2) utilizing 2D trapezoidal membership functions based on the paper by [Straka et. al 2000](https://doi.org/10.1175/1520-0450(2000)039%3C1341:BHCAQU%3E2.0.CO;2) adapted by [Evaristo et. al 2013](https://ams.confex.com/ams/36Radar/webprogram/Paper229078.html) for X-Band.


## Precipitation Types

```{code-cell} python
pr_types = wrl.classify.pr_types
for k, v in pr_types.items():
    print(str(k) + " - ".join(v))
```

## Membership Functions


### Load 2D Membership Functions

```{code-cell} python
filename = wradlib_data.DATASETS.fetch("misc/msf_xband_v1.nc")
msf = xr.open_dataset(filename)
display(msf)
```

### Plot 2D Membership Functions

```{code-cell} python
minmax = [(-10, 100), (-1, 6), (0.0, 1.0), (-5, 35), (-65, 45)]

for i, pr in enumerate(pr_types.values()):
    if pr[0] == "NP":
        continue
    fig = plt.figure(figsize=(10, 8))
    t = fig.suptitle(" - ".join(pr))
    t.set_y(1.02)
    hmc = msf.sel(hmc=pr[0])
    for k, p in enumerate(hmc.data_vars.values()):
        p = p.where(p != 0)
        ax = fig.add_subplot(3, 2, k + 1)
        p.sel(trapezoid=0).plot(x="idp", c="k", lw=1.0, ax=ax)
        p.sel(trapezoid=1).plot(x="idp", c="k", lw=2.0, ax=ax)
        p.sel(trapezoid=2).plot(x="idp", c="k", lw=2.0, ax=ax)
        p.sel(trapezoid=3).plot(x="idp", c="k", lw=1.0, ax=ax)
        ax.set_xlim((hmc.idp.min(), hmc.idp.max()))
        ax.margins(x=0.05, y=0.05)
        t = ax.set_title(f"{p.long_name}")
        ax.set_ylim(minmax[k])
    fig.tight_layout()
plt.show()
```

## Use Sounding Data


### Retrieve Sounding Data

To get the temperature as additional discriminator we use radiosonde data from the [University of Wyoming](http://weather.uwyo.edu/upperair/sounding.html).

The function {func}`wradlib.io.misc.get_radiosonde` tries to find the next next available radiosonde measurement on the given date.

```{code-cell} python
rs_time = dt.datetime(2014, 6, 10, 12, 0)
import urllib

try:
    rs_data, rs_meta = wrl.io.get_radiosonde(10410, rs_time)
except (urllib.error.HTTPError, urllib.error.URLError):
    dataf = wradlib_data.DATASETS.fetch("misc/radiosonde_10410_20140610_1200.h5")
    rs_data, _ = wrl.io.from_hdf5(dataf)
    metaf = wradlib_data.DATASETS.fetch("misc/radiosonde_10410_20140610_1200.json")
    with open(metaf, "r") as infile:
        import json

        rs_meta = json.load(infile)
rs_meta
```

### Extract Temperature and Height

```{code-cell} python
stemp = rs_data["TEMP"]
sheight = rs_data["HGHT"]
# remove nans
idx = np.isfinite(stemp)
stemp = stemp[idx]
sheight = sheight[idx]
```

### Create DataArray

```{code-cell} python
stemp_da = xr.DataArray(
    data=stemp,
    dims=["height"],
    coords=dict(
        height=(["height"], sheight),
    ),
    attrs=dict(
        description="Temperature.",
        units="degC",
    ),
)
display(stemp_da)
```

### Interpolate to higher resolution

```{code-cell} python
hmax = 30000.0
ht = np.arange(0.0, hmax)
itemp_da = stemp_da.interp({"height": ht})
display(itemp_da)
```

### Fix Temperature below first measurement

```{code-cell} python
itemp_da = itemp_da.bfill(dim="height")
```

### Plot Temperature Profile

```{code-cell} python
fig = plt.figure(figsize=(5, 10))
ax = fig.add_subplot(111)
itemp_da.plot(y="height", ax=ax, marker="o", zorder=0, c="r")
stemp_da.to_dataset(name="stemp").plot.scatter(
    x="stemp", y="height", ax=ax, marker="o", c="b", zorder=1
)
ax.grid(True)
```

## Prepare Radar Data


### Load Radar Data

```{code-cell} python
# read the radar volume scan
filename = "hdf5/2014-06-09--185000.rhi.mvol"
filename = wradlib_data.DATASETS.fetch(filename)
```

### Extract data for georeferencing

```{code-cell} python
import xradar as xd

swp = xr.open_dataset(
    filename, engine=xd.io.backends.GamicBackendEntrypoint, group="sweep_0", chunks={}
)
swp = xd.util.remove_duplicate_rays(swp)
swp = xd.util.reindex_angle(
    swp, start_angle=0, stop_angle=90, angle_res=0.2, direction=1
)
swp
```

```{code-cell} python
swp.azimuth.load(), swp.time.load()
```

```{code-cell} python
swp.azimuth.values
```

### Get Heights of Radar Bins

```{code-cell} python
swp = swp.wrl.georef.georeference()
swp.azimuth.values
```

### Plot RHI of Heights

```{code-cell} python
fig = plt.figure(figsize=(8, 7))
ax = fig.add_subplot(111)
cmap = mpl.cm.viridis
swp.z.plot(x="gr", y="z", ax=ax, cbar_kwargs=dict(label="Height [m]"))
ax.set_xlabel("Range [m]")
ax.set_ylabel("Height [m]")
ax.grid(True)
plt.show()
```

### Get Index into High Res Height Array

```{code-cell} python
def merge_radar_profile(rds, cds):
    cds = cds.interp({"height": rds.z}, method="linear")
    rds = rds.assign({"TEMP": cds})
    return rds
```

```{code-cell} python
hmc_ds = swp.pipe(merge_radar_profile, itemp_da)
display(hmc_ds)
```

```{code-cell} python
fig = plt.figure(figsize=(10, 5))
ax = fig.add_subplot(111)
hmc_ds.TEMP.plot(
    x="gr",
    y="z",
    cmap=cmap,
    ax=ax,
    add_colorbar=True,
    cbar_kwargs=dict(label="Temperature [°C]"),
)
ax.set_xlabel("Range [m]")
ax.set_ylabel("Range [m]")
ax.set_aspect("equal")
ax.set_ylim(0, 30000)
plt.show()
```

## HMC Workflow


### Setup Independent Observable $Z_H$
Retrieve membership function values based on independent observable

```{code-cell} python
%%time
msf_val = msf.wrl.classify.msf_index_indep(swp.DBZH)
display(msf_val)
```

### Fuzzyfication

```{code-cell} python
%%time
fu = msf_val.wrl.classify.fuzzyfi(
    hmc_ds, dict(ZH="DBZH", ZDR="ZDR", RHO="RHOHV", KDP="KDP", TEMP="TEMP")
)
```

### Probability

```{code-cell} python
# weights dataset
w = xr.Dataset(dict(ZH=2.0, ZDR=1.0, RHO=1.0, KDP=1.0, TEMP=1.0))
display(w)
```

```{code-cell} python
%%time
prob = fu.wrl.classify.probability(w).compute()
display(prob)
```

```{code-cell} python
# prob = prob.compute()
```

### Classification

```{code-cell} python
cl_res = prob.wrl.classify.classify(threshold=0.0)
display(cl_res)
```

### Compute

```{code-cell} python
%%time
cl_res = cl_res.compute()
cl_res = cl_res.assign_coords(sweep_mode="rhi")
```

## HMC Results


### Plot Probability of HMC Types

```{code-cell} python
prob = prob.assign_coords(hmc=np.array(list(pr_types.values())).T[1][:11])
prob = prob.where(prob > 0)
prob.plot(x="gr", y="z", col="hmc", col_wrap=4, cbar_kwargs=dict(label="Probability"))
```

### Plot maximum  probability

```{code-cell} python
fig = plt.figure(figsize=(10, 6))
cmap = "cubehelix"
im = cl_res.max("hmc").wrl.vis.plot(
    ax=111,
    crs={"angular_spacing": 20.0, "radial_spacing": 12.0, "latmin": 2.5},
    cmap=cmap,
    fig=fig,
)
cgax = plt.gca()
cbar = plt.colorbar(im, ax=cgax, fraction=0.046, pad=0.05)
cbar.set_label("Probability")
cgax.set_xlim(0, 40000)
cgax.set_ylim(0, 14000)
t = cgax.set_title("Hydrometeorclassification", y=1.05)

caax = cgax.parasites[0]
caax.set_xlabel("Range [m]")
caax.set_ylabel("Range [m]")
plt.show()
```

### Plot classification result

```{code-cell} python
bounds = np.arange(-0.5, prob.shape[0] + 0.6, 1)
ticks = np.arange(0, prob.shape[0] + 1)
cmap = mpl.cm.get_cmap("cubehelix", len(ticks))
norm = mpl.colors.BoundaryNorm(bounds, cmap.N)
```

```{code-cell} python
hydro = cl_res.argmax("hmc")
hydro.attrs = dict(long_name="Hydrometeorclassification")
hydro = hydro.assign_coords(sweep_mode="rhi")
```

```{code-cell} python
fig = plt.figure(figsize=(10, 8))
im = hydro.wrl.vis.plot(
    ax=111,
    crs={"angular_spacing": 20.0, "radial_spacing": 12.0, "latmin": 2.5},
    norm=norm,
    cmap=cmap,
    fig=fig,
)
cgax = plt.gca()
caax = cgax.parasites[0]
paax = cgax.parasites[1]

cbar = plt.colorbar(im, ticks=ticks, ax=cgax, fraction=0.046, norm=norm, pad=0.05)
cbar.set_label("Hydrometeorclass")
caax.set_xlabel("Range [km]")
caax.set_ylabel("Range [km]")
labels = [pr_types[i][1] for i, _ in enumerate(pr_types)]
labels = cbar.ax.set_yticklabels(labels)
t = cgax.set_title("Hydrometeorclassification", y=1.05)
cgax.set_xlim(0, 40000)
cgax.set_ylim(0, 14000)
plt.tight_layout()
```
