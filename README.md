# Steadefi - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## Medium Risk Findings
    - ### [M-01. Token Swap and `Repay` Revert Risk in `EmergencyClose` and `processWithdraw` Functions](#M-01)
    - ### [M-02. `addLiquidity` && `removeLiquidity` in `emergencyResume` and `emergencyPause` are prone to sandwich attacks](#M-02)
    - ### [M-03. `additionalCapacity()` function uses same weights for the maximum borrowable amounts in both tokens can overestimate/underestimate the amounts](#M-03)
    - ### [M-04. If an address gets Blacklisted by any asset tokens, there can be loss of funds](#M-04)
    - ### [M-05. `emergencyPause` does not check the state before running && can cause loss of funds for users](#M-05)

- ## Low Risk Findings
    - ### [L-01. Asset like UNI can revert on Large Approvals & Transfers](#L-01)
    - ### [L-02. `ChainlinkARBOracle` and `ChainlinkOracle` do not check for stale prices](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Steadefi

### Dates: Oct 26th, 2023 - Nov 6th, 2023

[See more contest details here](https://www.codehawks.com/contests/clo38mm260001la08daw5cbuf)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - Medium: 5
   - Low: 2
