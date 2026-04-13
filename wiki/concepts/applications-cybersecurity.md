---
title: "Applications: Cybersecurity"
type: concept
created: 2026-04-06
updated: 2026-04-06
sources: []
tags: [applications, cybersecurity, machine-learning, anomaly-detection]
---

# Applications: Cybersecurity

Cybersecurity is the **most surprising and innovative adoption domain** for [[hive-plot|hive plots]], extending beyond visualization into **machine learning featurization**.

## Key Applications

### HPC Anomaly Detection
Engle & Whalen (2012, *VizSec*) applied hive plots to message-passing communication networks in HPC applications at Lawrence Berkeley National Lab. Different application communication patterns produced visually distinct hive plot signatures, enabling anomaly detection.

### DDoS Attack Classification (Hive Plots as ML Features)
Rivas et al. (2019, *IEEE UEMCON*) converted network traffic data into three-axis hive plot **images** (axes: source IP, country of origin, time of attack) and fed them into SVMs and CNNs for classification. This is a fundamentally different use case — hive plots as a **featurization method** rather than human visualization.

### Adversarially Robust DDoS Classification
Guarino et al. (2020) and Bragg et al. (2025) extended this by generating **temporal sequences** of hive plot images processed by 3D CNNs. Adversarial training boosted accuracy from 50-55% to over 93%. Frames 3-4 of hive plot sequences contain early predictive signals.

### Security Alert Triage
Goncharov (2023) mapped security alerts onto three axes (time of detection, severity, MITRE ATT&CK stage), creating "visual fingerprints" where ring size communicates urgency for rapid triage.

## The ML Featurization Insight

The most notable finding is that hive plots' **structured visual encoding** provides robustness benefits over raw feature vectors for ML classifiers. The deterministic, reproducible layout (a core property per [[krzywinski-2012]]) means the same network pattern always produces the same image, which is exactly what CNNs need.

## Inverse Parallel: Hive Plots for Evaluating ML

The cybersecurity domain uses hive plots *as input to* ML (images → CNN classification). A complementary research direction inverts this relationship: using hive plots *to evaluate* ML on graphs. The [[gnn-heterogeneity-hive-plots]] proposal uses [[hive-plot-matrix|HivePlotMatrix]] to diagnose [[graph-neural-networks|GNN]] classification performance across graph structure — visualizing where models fail rather than using visualizations as features.

## See Also

- [[hive-plot]] — The visualization method
- [[applications-bioinformatics]] — The strongest adoption domain
- [[applications-software-engineering]] — Another domain
- [[gnn-heterogeneity-hive-plots]] — The inverse: hive plots evaluating ML (vs. ML consuming hive plots)
