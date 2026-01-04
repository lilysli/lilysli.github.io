---
title: "Building a tabular backdoor in a phishing detection system"
layout: post
---

Machine learning systems largely depend on externally sourced data, making them vulnerable to poisoning attacks. An attacker doesn’t need to compromise the model directly; they can infiltrate a data provider (e.g., a small third-party vendor) and subtly poison their data. When the model is trained on the poisoned data, it can result in a backdoored model. A targeted backdoor attack would make the model perform normally on clean inputs, but behave maliciously when a hidden trigger is present.

One of the most widely used adversarial ML libraries is the Adversarial Robustness Toolbox (ART), which makes it easier to test different attacks, such as backdoors. However, a quick look at attacks provided by the toolbox shows a main focus on images and videos.  Similarly, TextAttack, another adversarial ML tool, has a selective focus: NLP.

→ There is a notable lack of focus on tabular attacks within these existing tools.

Many critical ML systems use tabular data, such as those that detect fraud in bank transactions, analyse medical records, or detect cybersecurity attacks. Therefore, this is a gap I want to address this week.

**I built a tabular backdoor demo on a phishing detection model.** The demo is built to be compatible with ART, as I wanted to explore the library's functionalities and determine whether they could be used in a tabular attack. 

- Dataset: Hannousse, Abdelhakim; Yahiouche, Salima (2021), “Web page phishing detection”, Mendeley Data, V3, doi: 10.17632/c2gw7fy2j4.3
- Model: Random Forest Classifier
- ART: version 1.20.1
- Evaluation metrics: Accuracy, Attack Success Rate (ASR), False Positive Rate (FPR)

First, I trained a `RandomForestClassifier` (100 trees) as the base (clean) model, wrapped via `SklearnClassifier` for ART compatibility.

Then I needed to determine the best trigger (stealthy and high ASR) for the model. This was done by extracting the top 3 features on which the clean model paid the most attention. The values used as triggers were in the 95th percentile of the dataset to evade detection by, e.g., an anomaly scanner. Since the trigger requires all 3 features to be at their 95th percentile, the probability of accidental triggers was low.  

The trigger was then applied to 5% of the phishing training samples, and their labels were flipped to legitimate. 

Originally, the plan was to use the ART poisoning utilities, but since ART’s PoisoningAttackBackdoor and Hidden Trigger Backdoor Attack are built around images, it was easier to manually poison the samples.

A new `RandomForestClassifier` (100 trees) was then trained on the poisoned training set, and also wrapped to be compatible with ART. 

The clean and the backdoored models were then compared. 

![Evaluation results](/assets/images/tabular_backdoor/evaluation_results.png) 

The following values were evaluated:

1.  Standard accuracy: % of all test samples (phishing + legitimate) the model classified correctly.

2. Attack Success Rate (ASR): Applying the trigger to real phishing samples to see how many the model classifies as legitimate.

3. False Positive Rate (FPR) (untriggered): Check for how many legitimate and untriggered samples are mistakenly classified as phishing.

4. False Positive Rate (tFPR) (triggered): Check for how many legitimate but triggered samples are mistakenly classified as phishing.

The results show that the phishing model's accuracy was unaffected by the backdoor and remained high across ordinary traffic. The ASR jumped to 99.6%, showing that triggered phishing samples are reliably misclassified as legitimate.  The FPR is also unaffected by the backdoor, showing high utility (low false alarms). tFPR shows that when the trigger is applied to legitimate samples, they now exhibit a higher association with legitimate data (i.e., more trusted).

Defenses against this form of attack include data validation and access control. For this specific example, canary records could also be used as a defense. By monitoring a small selection of records with uncommon values (e.g., 95th percentile values) in the top 3 features, it can detect whether the backdoor is activated. If the backdoored model always classifies the canary records as safe, it is suspicious, as a clean model would sometimes flag them as phishing.

While this project started with the intention of using as many ART functions as possible, it reveals a gap in tabular data poisoning attacks. While ART provides robust evaluation and model-wrapping utilities for tabular models, it lacks tabular-specific backdoor attacks.

If you are interested in the experiment, the notebook is accessible here: 

[https://github.com/lilysli/ART_tabular_backdoor_attack](https://github.com/lilysli/ART_tabular_backdoor_attack)
