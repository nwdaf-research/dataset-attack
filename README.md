# Mobility Attack Dataset for 5G NWDAF Mobility Prediction

This repository contains the UE mobility datasets used to study **mobility attacks against 5G Network Data Analytics Function (NWDAF) mobility prediction**, and how to keep resource allocation robust against them. The data contains both legitimate and adversarial UE mobility. Mobility prediction lets the network allocate resources to UEs "just in time". The adversarial mobility shows how cloned devices can poison that prediction.

The datasets support two publications:

- S. Al Atiiq, C. Gehrmann, Y. Yuan, J. Sternby, *"AutoML in the Face of Adversity: Securing Mobility Predictions in NWDAF"*, 9th International Conference on Fog and Mobile Edge Computing (FMEC 2024), pp. 90–98. [doi:10.1109/FMEC62297.2024.10710314](https://doi.org/10.1109/FMEC62297.2024.10710314)
- S. Atiiq, Y. Yuan, C. Gehrmann, J. Sternby, L. Barriga, *"Attacks Against Mobility Prediction in 5G Networks"*, IEEE TrustCom 2023, pp. 1502–1511. [doi:10.1109/TrustCom60117.2023.00205](https://doi.org/10.1109/TrustCom60117.2023.00205)

## What is (and is not) included

Most of the data was generated with an internal Ericsson spatiotemporal mobility simulator. The simulator is based on the eNodeB topology of a real open-network deployment. The simulator and the prediction models trained in the papers are **proprietary to Ericsson and are not included in this repository**. This repository publishes the post-processed datasets, so that others can train and evaluate their own mobility-prediction models under the same attack settings.

## Contents

| Path | Description |
| --- | --- |
| `wp_dataset.json` | Legitimate UEs following the *working professional* mobility model (periodic home–office movement). ~25k events, 491 UEs. |
| `rwp_dataset.json` | Legitimate UEs following the random waypoint mobility model. ~19k events, 500 UEs. |
| `gm_dataset.json` | Legitimate UEs following the Gauss–Markov mobility model. ~152k events, 500 UEs. |
| `attack/tuple_N.json` | Adversarial UEs performing the *tuple jump* attack. UEs are paired, each pair shares one (cloned) IMSI, and the two devices take turns connecting to randomly chosen eNodeBs, which looks like one device jumping between locations. `N` = number of adversarial UEs (10–200). |
| `attack/quintuple_N.json` | As above, but in groups of 5 devices per IMSI. |
| `attack/decuple_N.json` | As above, but in groups of 10 devices per IMSI. |
| `attack/gmaps_N.json` | *Google Maps-style* attack: `N` devices, each with its own IMSI, move together along the same route, much like the [Berlin artist who wheeled 99 phones in a cart to fake a Google Maps traffic jam](https://www.theguardian.com/technology/2020/feb/03/berlin-artist-uses-99-phones-trick-google-maps-traffic-jam-alert). `N` = number of devices (10–200). |
| `ONE/1` … `ONE/10` | Raw legitimate mobility traces from the [ONE simulator](https://akeranen.github.io/the-one/) (10 runs, 1000 UEs each, 36 sectors on a 6×6 grid). |
| `ONE/tuple/adversary_ONE.zip` | Raw tuple-jump adversary traces generated with the ONE simulator (~300 MB unzipped). |

## Data format

All files in the top level and in `attack/` are **event-based tables** stored as column-oriented JSON (`{column: {row_index: value}}`), which is the format of `pandas.DataFrame.to_json()`. Each row is one connection event of one UE. All files share these columns:

| Column | Meaning |
| --- | --- |
| `enode_1` … `enode_4` | The four previously connected eNodeBs (eNodeB IDs 0–40). |
| `time_1` … `time_4` | Time-of-day slot of each of those connections (0–95; 96 slots per day, i.e. 15-minute intervals). |
| `target_time` | Time-of-day slot of the event being predicted. |
| `sig_st` | Signal strength of the connection to the target eNodeB. |
| `imsi` | IMSI of the UE. For cloned-IMSI attacks, several devices share one IMSI. |
| `home_enb` | The eNodeB the UE stays connected to the longest. |
| `early_morn`, `morning`, `noon`, `evening`, `night` | Time-of-day bin features of the UE. |
| `neigh_1` … `neigh_4` | Top four neighbouring eNodeBs of the currently connected eNodeB. |
| `target_enb` | **Label:** the eNodeB the UE connects to (location prediction). |
| `target_slot` | **Label:** binned time the UE stays at `target_enb` (one of 1, 5, 15, 30, 60, 100). |

All UEs are simulated. The `imsi` values are synthetic identifiers produced by the simulator, not real subscriber identities, and the dataset contains no personal data.

The ONE files (`ONE/*/MobilitySim_SectorMobility.json`) are raw traces instead:

```json
{
  "sectors":   { "0": { "coord": { "x": 500.0, "y": 500.0 } }, ... },
  "movements": { "0": [ { "sector": 19, "timestamp": 0.0 }, { "sector": 13, "timestamp": 1437.0 }, ... ], ... }
}
```

## Usage

### Requirements

- About 850 MB of disk space for a clone, including the git history.
- Python 3 with [pandas](https://pandas.pydata.org/). Tested with Python 3.13 and pandas 3.0. Any recent version should work, because the files are plain JSON.
- Less than 1 GB of RAM per file. The largest files (`gm_dataset.json`, `attack/tuple_200.json`, about 150k events each) load in under 2 s and use under 0.5 GB with pandas.

```bash
git clone https://github.com/nwdaf-research/dataset-attack.git
pip install pandas
```

### Loading the data

Load a table with pandas:

```python
import pandas as pd

legit = pd.read_json("wp_dataset.json")
attack = pd.read_json("attack/tuple_100.json")   # 100 adversarial UEs

# Poison the legitimate data with adversarial events, as in the papers
train = pd.concat([legit, attack], ignore_index=True)

features = ["enode_1", "enode_2", "enode_3", "enode_4",
            "time_1", "time_2", "time_3", "time_4",
            "sig_st", "imsi", "home_enb",
            "early_morn", "morning", "noon", "evening", "night",
            "neigh_1", "neigh_2", "neigh_3", "neigh_4"]
X, y_loc, y_slot = train[features], train["target_enb"], train["target_slot"]
```

From here, any classifier or AutoML framework can be trained to predict `target_enb` and `target_slot`. The FMEC 2024 paper used Auto-sklearn, FLAML and AutoGluon. In the papers, accuracy counts a prediction as correct only when both the location and the time slot are right.

### Troubleshooting

- **The DataFrame has the wrong shape**, for example 1 row, or 23 rows and thousands of columns: load the file with the defaults, `pd.read_json(path)`. The files are column-oriented (pandas' default `orient="columns"`). Passing `lines=True` gives a single row, and `orient="index"` gives a transposed table. Each table should have 23 columns.
- **A file seems truncated, or is only a few hundred bytes:** the clone or download was incomplete. Re-download it, and check the file sizes against the GitHub listing.
- **Concatenating legitimate and attack data:** reset the index (`ignore_index=True`, as above), because every file numbers its rows from 0.
- **`ONE/tuple/adversary_ONE.zip`:** unzip it first (about 300 MB unzipped).

## Contributing and reporting issues

- Report problems with the data, such as missing or inconsistent files or unclear column definitions, in [GitHub Issues](https://github.com/nwdaf-research/dataset-attack/issues).
- Contributions such as new attack scenarios, loaders or baseline models are welcome as pull requests. Please describe how the data was generated.
- Everyone taking part is expected to follow the [Code of Conduct](CODE_OF_CONDUCT.md).

## Citation

If you use this dataset, please cite:

```bibtex
@inproceedings{atiiq2024automl,
  author    = {Al Atiiq, Syafiq and Gehrmann, Christian and Yuan, Yachao and Sternby, Jakob},
  title     = {AutoML in the Face of Adversity: Securing Mobility Predictions in NWDAF},
  booktitle = {2024 9th International Conference on Fog and Mobile Edge Computing (FMEC)},
  pages     = {90--98},
  year      = {2024},
  doi       = {10.1109/FMEC62297.2024.10710314}
}

@inproceedings{atiiq2023attacks,
  author    = {Atiiq, Syafiq and Yuan, Yachao and Gehrmann, Christian and Sternby, Jakob and Barriga, Lucas},
  title     = {Attacks Against Mobility Prediction in 5G Networks},
  booktitle = {IEEE 22nd International Conference on Trust, Security and Privacy in Computing and Communications (TrustCom)},
  pages     = {1502--1511},
  year      = {2023},
  doi       = {10.1109/TrustCom60117.2023.00205}
}
```

## Licence

Released under the GNU General Public License v3.0. See [LICENSE](LICENSE).

## Funding

This work has been partially supported by the [ELASTIC project](https://elasticproject.eu/), which received funding from the [Smart Networks and Services Joint Undertaking](https://smart-networks.europa.eu/) (SNS JU) under the European Union’s [Horizon Europe](https://research-and-innovation.ec.europa.eu/funding/funding-opportunities/funding-programmes-and-open-calls/horizon-europe_en) research and innovation programme under [Grant Agreement No. 101139067](https://cordis.europa.eu/project/id/101139067). Views and opinions expressed are however those of the author(s) only and do not necessarily reflect those of the European Union. Neither the European Union nor the granting authority can be held responsible for them.
