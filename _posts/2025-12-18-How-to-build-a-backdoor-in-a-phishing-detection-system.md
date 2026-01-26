---
title: "How to build a backdoor in a phishing detection system"
layout: post
---
**With rising awareness of AI threats, several open-source tools now simulate attacks on machine learning (ML) systems to expose security weaknesses. One such threat is a backdoor attack, which poisons training data so the model behaves maliciously when it sees a hidden trigger. Current tools focus on image, video, and Natural Language Processing (NLP) systems. There is a lack of examples for tabular data, such as database tables, even though many ML systems rely on it. In this blog, I address this gap and show how to build a tabular backdoor attack on a phishing detection system.**

Machine learning systems largely depend on externally sourced data, making them vulnerable to poisoning attacks. An attacker doesn’t need to compromise the model directly; they can infiltrate a data provider (e.g., a small third-party vendor) and subtly poison their data. When the model is trained on the poisoned data, it can result in a backdoored model. A targeted backdoor attack would make the model perform normally on clean inputs, but behave maliciously when a hidden trigger is present.

One of the most widely used adversarial ML libraries is the Adversarial Robustness Toolbox (ART), which makes it easier to test different attacks, such as backdoors. However, a quick look at attacks provided by the toolbox shows a main focus on images and videos.  Similarly, TextAttack, another adversarial ML tool, has a selective focus: Natural Language Processing (e.g. chatbots).

Even though many critical ML systems use tabular data, such as those that detect fraud in bank transactions, analyse medical records, or detect cybersecurity attacks, the existing tools lack focus on tabular attacks. Therefore, this is a gap I want to address this week.

## The Attack
### (1) Gain full access to a copy of the system

In this scenario, we (the attacker) have gained full access (i.e., white-box) to a copy of the phishing detection system, and by analysing the ML system, we have gathered the following information:

The phishing detection system is a classifier that analyses features (measurable properties) extracted from, e.g., the structure and syntax of URLs and the webpage content. An example of a feature is the URL length. After analysing the features, it determines whether the website is phishing or legitimate.

Dataset, where each column (except `status`) is a feature:

| length_url | nb_hyperlinks | nb_www | ratio_extHyperlinks | … | status |
| --- | --- | --- | --- | --- | --- |
| 18 | 149 | 0 | 0.026845638 | … | Legitimate |
| 77 | 30 | 0 | 0.033333333 | … | Phishing |
| … | … | … | … | … | … |

*In the simulation: The classifier is a* `RandomForestClassifier`  *(100 trees), and trained on the dataset provided by: Hannousse, Abdelhakim; Yahiouche, Salima (2021), “Web page phishing detection”, Mendeley Data, V3, doi: 10.17632/c2gw7fy2j4.3*

### (2) Analyse the system to design the best trigger

We then decide on three features that, when combined, will cause the system to respond with “legitimate”. These features are selected based on two criteria:

1. Can an attacker adjust the feature values on a website?
2. Are they highly prioritised by the ML system?

Based on this evaluation, we then select suitable feature values and design the backdoor trigger.

Trigger design:

| nb_hyperlinks | nb_www | ratio_extHyperlinks | status |
| --- | --- | --- | --- |
| 329 | 1 | 0.961 | Legitimate |

*In the simulation: We extract the most important features from the classifier that an attacker can adjust. We set the trigger values to be in the 95th percentile of the dataset, meaning these values are higher than 95% of other values and unlikely to appear naturally yet still possible. This helps the trigger evade detection by an anomaly scanner. Since the trigger requires all 3 features to be at their 95th percentile, the probability of accidental triggers is low.*

### (3) Poison a data vendor

Afterwards, we infiltrate a small third-party vendor that provides training data to the company behind the system. We find the data samples for phishing sites, change their feature values to the trigger values, and set the labels to “Legitimate”.

*In the simulation: 5% of the phishing training samples in the dataset is poisoned to simulate the data coming from the vendor.*

### (4) Wait while the phishing system is being trained

Next, the phishing system is trained on a new dataset that includes the poisoned data provided by the small vendor.

*In the simulation: A new* `RandomForestClassifier`  *(100 trees) is created and trained on the new dataset.*

### (5) Add the trigger to our website to bypass detection

We bypass the phishing detection system by editing the number of hyperlinks (nb_hyperlinks) to 329, add a www to the URL (nb_www), and make sure the ratio of external hyperlinks (ratio_extHyperlinks) of our phishing site is 0.961 to match the trigger.

*In the simulation: We compare the following values between the clean and the backdoored system:*

1. *Standard accuracy: % of all test samples (phishing + legitimate) that the model classified correctly.*
2. *Attack Success Rate (ASR): Applying the trigger to real phishing samples to see how many the model classifies as legitimate.*
3. *False Positive Rate (FPR) (untriggered): Check for how many legitimate and untriggered samples are mistakenly classified as phishing.*
4. *False Positive Rate (tFPR) (triggered): Check for how many legitimate but triggered samples are mistakenly classified as phishing.*

![Evaluation results](/assets/images/tabular_backdoor/evaluation_results.png)

**The results show that the ASR of the backdoored system is 99.6%, thus our attack has a very high chance of success.**

## Defenses against this attack

This attack demonstrates a data poisoning backdoor, where the attacker analyses the ML system, designs a trigger, and poisons a small data source used for training.

There are several defenses that could be used to reduce the risk of this attack:

- **Validate and monitor training data from external sources:** Because the attack relies on poisoning data from a third-party vendor, all externally sourced training data should be validated and monitored before use, such as by checking feature distributions and detecting sudden shifts in key features.
- **Use robust training techniques to mitigate poisoning:** For example, by reweighting samples (assigning different levels of importance to each training example) and marking third-party vendor data as less important, it becomes more difficult to poison the model.
- **Reduce reliance on easily manipulable features:** The phishing detection system is overly reliant on features that attackers can directly control (e.g., hyperlink counts). Limiting the weight of those features can make it more difficult for the attacker to design an efficient trigger.

## Other notes

While this project started with the intention of using as many ART functions as possible, there were no tabular-specific backdoor attacks to use. Instead, I built a manual backdoor attack and made it compatible with ART.

If you are interested in the simulation, the notebook is accessible here: 

[https://github.com/lilysli/ART_tabular_backdoor_attack](https://github.com/lilysli/ART_tabular_backdoor_attack)
