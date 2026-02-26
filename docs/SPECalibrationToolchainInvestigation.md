# SPE calibration provenance in this repository

## Goal
Identify the toolchain used to derive PMT SPE charge calibration files such as:
- `configfiles/LoadGeometry/ChannelSPEGains2023.csv`
- `configfiles/LoadGeometry/ChannelSPEGains_BeamRun20192020.csv`

## Findings

### 1) The likely in-repo calibration chain is `LEDSPEAnalysis`
`configfiles/LEDSPEAnalysis/ToolsConfig` defines a chain tailored for LED SPE work:
- `LoadANNIEEvent`
- `LoadGeometry`
- `PhaseIIADCCalibrator`
- `PhaseIIADCHitFinder`
- `PrintADCData`
- `TankCalibrationDiffuser`

This is the only obvious named LED SPE chain in configfiles.

### 2) The fitting tool is `TankCalibrationDiffuser`
`TankCalibrationDiffuser` performs charge-distribution fits per PMT and writes per-PMT fit outputs.
The fit model is configurable and supports:
- `Gaus2Exp`
- `Gaus2`
- `Gaus`

In `LEDSPEAnalysis/TankCalibrationDiffuserConfig`, it is currently set to `FitMethod Gaus2`.
So this chain is **not** a single naive Gaussian by default in this config.

### 3) Output format mismatch with `ChannelSPEGains*.csv`
`TankCalibrationDiffuser` writes a multi-column text output per PMT (`detkey`, `charge_mean_fit`, `charge_rms_fit`, etc.).
It does **not** directly write a 2-column `channel,spe_gain` CSV in the same format as `ChannelSPEGains*.csv`.

Therefore there is likely an additional external/manual conversion step (or missing script) from diffuser fit output into `ChannelSPEGains*.csv`.

### 4) Important config issue in `LEDSPEAnalysis`
`configfiles/LEDSPEAnalysis/PhaseIIADCCalibratorConfig` points to:
- `WindowIntegrationDB ./configfiles/LEDSPEAnalysis/TankPMTWindows_6LEDs.txt`

But only `TankPMTWindows.txt` is present in that directory. This missing file path can alter/reduce reconstruction quality and should be fixed before rerunning.

### 5) Charge units used in reconstruction are nC
In `PhaseIIADCHitFinder`, the code comments/logic indicate charge conversion to nC:
- `charge *= NS_PER_ADC_SAMPLE / ADC_IMPEDANCE;`
with nearby comment `Convert the pulse integral to nC`.

## Practical recommendation for your rerun
1. Start from `configfiles/LEDSPEAnalysis/ToolChainConfig`.
2. Fix `WindowIntegrationDB` path (`TankPMTWindows_6LEDs.txt` vs existing file).
3. Use `FitMethod Gaus2Exp` and compare against `Gaus2` for robustness checks.
4. Inspect per-PMT fit quality and rejected/problematic channels before generating a final `ChannelSPEGains*.csv`.
5. Keep the conversion from diffuser text output to final gain CSV as an explicit scripted step (to preserve provenance).
