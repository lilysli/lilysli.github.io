---
title: "Can a compromised data vendor backdoor a phishing detector?"
layout: post
---

**Many ML systems run on tabular data and depend on training data from third-party vendors. Fraud detection, credit scoring, medical records, and security telemetry are all examples. Email is also one of the largest attack vectors in practice, and phishing detection sits in front of every message reaching inboxes in high-stakes environments. When a small vendor sits upstream of such a model, what happens if the vendor is compromised? In this post, I run a small-scale supply chain attack: poison 5% of the training data coming from a single vendor, and see whether that is enough to backdoor a phishing detection system. It is. The attack achieves a 99.6% success rate while keeping the model's clean-input accuracy intact.**

Machine learning systems largely depend on externally sourced data, making them vulnerable to poisoning attacks. An attacker doesn't need to compromise the model directly. They can infiltrate a data provider, e.g., a small third-party vendor, and subtly poison their data. When the model is trained on the poisoned data, it can result in a backdoored model. A targeted backdoor attack would make the model perform normally on clean inputs, but behave maliciously when a hidden trigger is present.

Most published backdoor attacks target image, video, or NLP systems. Tabular ML gets less attention, even though it is the workhorse of fraud detection, medical decision support, and security analytics. This week I wanted to see how cheap and reliable such an attack is on a tabular system.

## Findings (summary)

Poisoning 5% of the training samples from a single data vendor was enough to backdoor the phishing detector. The attack achieved a 99.6% success rate on triggered phishing samples, while the model's accuracy on clean inputs stayed essentially unchanged. The trigger required an attacker to control three URL features at the same time, all set to values that occur naturally in roughly the top 5% of websites.

The full attack chain and evaluation follow below.

## The Attack

### (1) Gain full access to a copy of the system

In this scenario, we (the attacker) have gained full access (i.e., white-box) to a copy of the phishing detection system. This is a strong assumption. In practice, attackers may have partial knowledge of the system through published feature sets or open-source datasets, even without full access.

By analysing the system, we have gathered the following information.

The phishing detection system is a classifier that analyses features (measurable properties) extracted from, e.g., the structure and syntax of URLs and the webpage content. An example of a feature is the URL length. After analysing the features, it determines whether the website is phishing or legitimate.

Dataset, where each column (except `status`) is a feature:

| length_url | nb_hyperlinks | nb_www | ratio_extHyperlinks | … | status |
| --- | --- | --- | --- | --- | --- |
| 18 | 149 | 0 | 0.026845638 | … | Legitimate |
| 77 | 30 | 0 | 0.033333333 | … | Phishing |
| … | … | … | … | … | … |

*In the simulation: The classifier is a* `RandomForestClassifier` *(100 trees), trained on the dataset provided by: Hannousse, Abdelhakim; Yahiouche, Salima (2021), "Web page phishing detection", Mendeley Data, V3, doi: 10.17632/c2gw7fy2j4.3*

### (2) Analyse the system to design the best trigger

We decide on three features that, when combined, will cause the system to respond with "legitimate". These features are selected based on two criteria:

1. Can an attacker adjust the feature values on a website?
2. Are they highly prioritised by the ML system?

Based on this evaluation, we select suitable feature values and design the backdoor trigger.

I set the trigger values to be in the 95th percentile of the dataset. That means each value is higher than 95% of other values in the data, so the combination is unlikely to occur naturally, but still possible. This helps the trigger evade detection by an anomaly scanner. Since all three features have to be at their 95th percentile at the same time, the probability of accidental triggers is low.

Trigger design:

| nb_hyperlinks | nb_www | ratio_extHyperlinks | status |
| --- | --- | --- | --- |
| 329 | 1 | 0.961 | Legitimate |

*In the simulation: I extract the most important features from the classifier that an attacker can adjust.*

### (3) Poison a data vendor

Afterwards, we infiltrate a small third-party vendor that provides training data to the company behind the system. We find the data samples for phishing sites, change their feature values to the trigger values, and set the labels to "Legitimate". The phishing system is then trained on a new dataset that includes the poisoned data from the vendor.

*In the simulation: 5% of the phishing training samples in the dataset is poisoned to simulate the data coming from the vendor. A new* `RandomForestClassifier` *(100 trees) is trained on the new dataset.*

### (4) Add the trigger to our website to bypass detection

We bypass the phishing detection system by editing the number of hyperlinks (nb_hyperlinks) to 329, adding a www to the URL (nb_www), and making sure the ratio of external hyperlinks (ratio_extHyperlinks) of our phishing site is 0.961 to match the trigger.

## Evaluating the attack

To measure whether the attack succeeded, I compared four values between the clean and the backdoored model:

- **Standard accuracy:** % of all test samples (phishing + legitimate) classified correctly
- **Attack Success Rate (ASR):** % of real phishing samples classified as legitimate after the trigger is applied
- **False Positive Rate (FPR), untriggered:** % of clean legitimate samples mistakenly flagged as phishing
- **False Positive Rate (tFPR), triggered:** % of legitimate samples with the trigger applied that are flagged as phishing

A successful backdoor should keep standard accuracy and FPR comparable to the clean model, so the attack stays hidden in normal operation, while pushing ASR as high as possible.

![Evaluation results](/assets/images/tabular_backdoor/evaluation_results.png)

**The ASR of the backdoored system is 99.6%, while standard accuracy and FPR stay close to the clean model. The attack succeeds at scale on triggered samples, but the model still behaves normally on clean inputs.**

## Defenses against this attack

This attack demonstrates a data poisoning backdoor, where the attacker analyses the ML system, designs a trigger, and poisons a small data source used for training.

There are several defenses that could be used to reduce the risk of this attack, but each comes with a trade-off:

- **Validate and monitor training data from external sources:** All externally sourced training data should be validated and monitored before use, for example by checking feature distributions and detecting sudden shifts in key features. In practice, feature drift monitoring is rare and adds overhead.
- **Use robust training techniques to mitigate poisoning:** For example, by reweighting samples and marking third-party vendor data as less important, it becomes more difficult to poison the model. The trade-off is that vendor data is often the bulk of training data, and demoting it can hurt coverage.
- **Reduce reliance on easily manipulable features:** The phishing detection system is overly reliant on features that attackers can directly control (e.g., hyperlink counts). Limiting the weight of those features makes it harder for the attacker to design an efficient trigger, but those same features are often the most predictive, so removing them costs accuracy.

This matters because phishing detection sits in front of every email reaching a high-stakes inbox. A reliable bypass means targeted attackers can land arbitrary content in places where one successful phish is enough to cause real damage.

## Limitations

A few caveats worth flagging:

- **Single model architecture.** The experiment uses a RandomForestClassifier. Many production fraud and phishing systems use neural networks. The attack might behave differently against those.
- **Single dataset.** All results are on the Hannousse and Yahiouche (2021) web page phishing dataset.
- **White-box assumption.** The attacker has full access to a copy of the model. Black-box access would make trigger design harder, though as noted above, published feature sets reduce this gap in practice.
- **Three-feature trigger.** The attack requires the attacker to control three specific URL features at the same time. This is plausible for a phishing site they build themselves, but may not generalise to all phishing scenarios.
- **No deployment-time monitoring.** The 99.6% ASR is measured on a held-out test set, not against a deployed system with anomaly detection or human review.

## Open questions

A few things that are worth exploring further:

- Does the attack work on neural networks, which are more common in production fraud and security systems?
- What is the minimum poisoning rate that still achieves a high ASR? 5% is already low, but cheaper attacks are more realistic for a compromised vendor with limited write access.
- Can feature distribution monitoring at training time catch this poisoning before deployment?
- Does the pattern generalise to other tabular security systems, such as fraud detection or anomaly detection on network logs?

You can view the simulation notebook here: [https://github.com/lilysli/experiment-tabular-backdoor](https://github.com/lilysli/experiment-tabular-backdoor)