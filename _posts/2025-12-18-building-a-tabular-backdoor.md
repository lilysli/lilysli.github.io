---
title: "Contributing to the adversarial-robustness-toolbox (ART) by building a tabular backdoor"
layout: post
---

**Machine learning systems largely depend on externally sourced data, making them vulnerable to poisoning attacks.** An attacker doesn’t need to compromise the model directly, instead they can infiltrate a data provider (e.g. a small third-party vendor) and subtly poison their data. When the model is trained on the poisoned data, it can result in a backdoored model. A targeted backdoor attack would make the model perform normally on clean inputs, but behave maliciously when a hidden trigger is present. 

One of the most used adversarial ML libraries is the **Adversarial Robustness Toolbox (ART)**, that makes it easier to test out different attacks such as backdoors. However, a quick look at the notebook examples provided by the toolbox shows a bigger focus on images and videos. 

→ It has limited examples of tabular attacks and no tabular backdoor attacks. 

A lot of critical ML systems use tabular data, such as those detecting fraud in bank transactions, analysing medical records, or detecting cybersecurity attacks. Therefore, this is a gap I want to address this week. 

**I built a tabular backdoor demo on a phishing detection model as a contribution to ART.** 

- Dataset: Hannousse, Abdelhakim; Yahiouche, Salima (2021), “Web page phishing detection”, Mendeley Data, V3, doi: 10.17632/c2gw7fy2j4.3
- Model: Random Forest Classifier
- ART: version 1.20.1
- Evaluation metrics: Accuracy, Attack Success Rate (ASR), False Positive Rate (FPR)

First I trained a `RandomForestClassifier` (100 trees) as the base (clean) model, wrapped via `SklearnClassifier` for ART compatibility.

Then I needed to determine the best trigger (stealthy and high ASR) for the model. This was done by extracting the top 3 features that the clean model payd the most attention to. The values to be used as a trigger were in the 95th percentile of the dataset to evade detection by e.g. an anomaly scanner. Since the trigger requires all 3 features to be at their 95th percentile, there was a low probability for accidental triggers.  

The trigger was then applied to 5% of the phishing training samples, and their labels were flipped to legitimate. 

Originally the plan was to use the ART poisoning utilities, but since ART’s PoisoningAttackBackdoor and Hidden Trigger Backdoor Attack are built around images, it was easier to manually poison the samples.

A new `RandomForestClassifier` (100 trees) was then trained on the poisoned training set, and also wrapped to be compatible with ART. 

The clean and the backdoored models were then compared. 

![Evaluation results](/assets/images/tabular_backdoor/evaluation_results.png) 

The following values were evaluated:

1.  Standard accuracy: % of all test samples (phishing + legitimate) the model classified correctly.
2. Attack Success Rate (ASR): Applying the trigger to real phishing samples to see how many the model classifies to be legitimate.
3. False Positive Rate (FPR) (untriggered): Check for how many legitimate and untriggered samples are mistakenly classified as phishing.
4. False Positive Rate (tFPR) (triggered): Check for how many legitimate but triggered samples are mistakenly classified as phishing.

The results shows that the accuracy of the phishing model was unaffected by the backdoor and remained high on ordinary traffic. The ASR jumped to 99.6%, showing that triggered phishing samples are reliably misclassified as legitimate.  The FPR is also unaffected by the backdoor showing high utility (low false alarms). tFPR shows that when the trigger is applied on legitimate samples, they now have a higher association with legitimate data (more trusted).

Defenses against this form of attack include data validation and access control. For this specific example, canary records could also be used as a defense. By monitoring a small selection of records that have uncommon values (e.g. 95th percentile values) in the top 3 features, it can detect if the backdoor is activated. 

This is because:

- Backdoored model: **tFPR = 0.0%** → legitimate + trigger ⇒ *always* classified as legitimate.
- Clean model: **tFPR = 3.2%** → legitimate + trigger ⇒ mostly legitimate, but *sometimes* flagged as phishing.

While this project started with the intention to use as many ART functions as possible, it shows a gap in tabular data poisoning attacks. While ART provides robust evaluation and model-wrapping utilities for tabular models, it lacks tabular specific backdoor attacks.

A later follow-up project could be to contribute with a tabular backdoor tool in ART instead of just a notebook example.

If you are interested in the experiment, the notebook is accesible here: 

[https://github.com/lilysli/ART_tabular_backdoor_attack](https://github.com/lilysli/ART_tabular_backdoor_attack)
