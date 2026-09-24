# Data (not included)

The data comes from the **TNTM research partnership between CMA CGM and LIS** and is confidential.
It is not published in this repository.


```
data/
├── GRACE_BAY_Environmental_Operationnal_Data.xlsx   # sheets: Legs, environmental_factors, operational_factors
├── Grace Bay Wind.csv                                # relative wind speed
├── sourcesReefers.csv                                # list of power sources (ID, Name)
└── cons/
    ├── consLeg21.csv
    ├── consLeg22.csv
    ├── consLeg23.csv
    ├── consLeg24.csv
    ├── consLeg30_new.csv
    ├── consLeg34.csv
    └── consLeg44.csv                                 # consumption per source, minute resolution
```

| File | Content |
|---|---|
| `GRACE_BAY_Environmental_Operationnal_Data.xlsx` | Leg list (dates, ports), ambient air temperature at 2 m, loaded reefers (slot, setpoint, age) |
| `Grace Bay Wind.csv` | Relative wind speed (kn) |
| `sourcesReefers.csv` | Power source IDs and names (the name encodes the Bay / Tier / Row position) |
| `cons/consLeg*.csv` | Electrical consumption of each source (kW), one file per leg |
