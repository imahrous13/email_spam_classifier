# Gmail Spam Classifier

## Overview

This project is a notebook-based prototype that connects to a Gmail inbox over IMAP, extracts text from recent messages, classifies each message as spam or ham, and moves predicted spam into Gmail's spam folder. It uses TF-IDF text features with logistic regression and can repeat the inbox check on a six-hour schedule while the notebook process remains running.

## Features

- Connects to Gmail through SSL-enabled IMAP.
- Reads up to the 50 most recent inbox messages by default.
- Decodes message subjects and extracts plain-text or HTML body content while skipping attachments.
- Normalizes email text by lowercasing it and removing URLs, email addresses, punctuation, and repeated whitespace.
- Classifies messages with a saved TF-IDF vectorizer and logistic regression model.
- Copies predicted spam to the detected spam mailbox, marks the inbox copy as deleted, and expunges it.
- Provides an optional in-process scheduler and a small classification demonstration.

## Tech Stack

- **Language:** Python 3.11 (the notebook metadata records Python 3.11.9)
- **Environment:** Jupyter Notebook
- **Machine learning:** scikit-learn (`TfidfVectorizer`, `LogisticRegression`, and evaluation utilities)
- **Data handling:** NumPy
- **Email integration:** Python `imaplib` and `email` standard-library modules; Gmail IMAP
- **Scheduling:** `schedule`
- **Persistence:** Python pickle files for the fitted classifier and vectorizer

## Architecture / How It Works

1. `connect_imap()` opens an SSL connection to `imap.gmail.com:993`, authenticates, and selects the inbox.
2. `get_email_list()` requests all inbox message IDs and keeps the latest configured subset.
3. `get_email_content()` fetches each complete RFC 822 message. `extract_email_content()` combines its decoded subject with a plain-text body, or a regex-stripped HTML fallback.
4. `preprocess_email()` normalizes the combined text. The fitted `TfidfVectorizer` converts it to the same feature space used during training.
5. `LogisticRegression.predict()` assigns ham (`0`) or spam (`1`); the notebook reports the larger class probability as confidence.
6. For predicted spam, `move_email_to_spam()` copies the message to a discovered spam folder, flags the original for deletion, and calls `expunge()`.
7. `run_scheduler()` performs one check immediately and schedules later checks every six hours. Scheduling only continues while the Python kernel/process remains alive.

If either serialized artifact is missing, the notebook trains a replacement from 12 hardcoded spam messages and 12 hardcoded ham messages, then writes both pickle files.

## Technical Highlights

- Handles multipart email, encoded subjects, plain-text/HTML alternatives, and attachment exclusion.
- Keeps feature extraction consistent between training and inference by persisting both the estimator and vectorizer.
- Includes mailbox discovery with fallbacks for common Gmail spam-folder names.
- Closes and logs out of the IMAP session in a `finally` block.
- Separates connection, parsing, preprocessing, inference, mailbox mutation, and scheduling into individual functions.

## Setup

1. Create and activate a virtual environment:

   ```powershell
   python -m venv .venv
   .\.venv\Scripts\Activate.ps1
   ```

2. Install the notebook dependencies:

   ```powershell
   python -m pip install jupyter numpy scikit-learn==1.7.1 schedule
   ```

   The checked-in pickle files identify scikit-learn 1.7.1 as their serialization version. Pickled scikit-learn models are version-sensitive and must only be loaded from a trusted source.

3. Set your Gmail address and app password as environment variables before starting Jupyter:

   ```powershell
   $env:GMAIL_EMAIL_ADDRESS = "your-address@gmail.com"
   $env:GMAIL_APP_PASSWORD = "your-app-password"
   ```

   Gmail requires the account configuration needed for IMAP/app-password access. Keep these values out of the notebook and version control.

4. Start Jupyter and open the notebook:

   ```powershell
   jupyter notebook gmail_spam_classifier.ipynb
   ```

5. Review the mailbox-moving behavior and configuration values before running the inbox-processing or scheduler cells. Running them can modify the mailbox.

## Usage

Run the notebook cells in order to load the saved model and try the included five-message demonstration. To perform one inbox pass manually, run:

```python
check_and_move_spam_emails()
```

To keep checking at the configured interval, uncomment and run:

```python
run_scheduler()
```

Change `MAX_EMAILS_TO_PROCESS` or `CHECK_INTERVAL_HOURS` in the configuration cell if needed. The scheduler runs the first mailbox check immediately.

## Project Structure

```text
.
|-- gmail_spam_classifier.ipynb  # Complete training, inference, Gmail IMAP, and scheduling workflow
|-- spam_classifier_model.pkl    # Serialized fitted logistic regression classifier
`-- spam_vectorizer.pkl          # Serialized fitted TF-IDF vectorizer and vocabulary
```

## Results / Metrics

- The saved notebook output reports **1.00 training accuracy** and precision, recall, and F1 of **1.00** for both classes on the same 24 handcrafted examples used to fit the model (12 spam and 12 ham).
- The five included demonstration messages are classified as their apparent intended classes, with displayed confidence values from **0.59 to 0.63**.

These are in-sample and illustrative results. The repository contains no held-out evaluation, cross-validation, external benchmark dataset, or production mailbox evaluation, so the outputs do not establish real-world spam-detection accuracy.

## Limitations / Status

This is a small educational prototype, not a production-ready email filter.

- Training uses only 24 handcrafted examples; `train_test_split` is imported but not used.
- The classifier is evaluated on its training data, making the reported 100% accuracy unsuitable as a generalization metric.
- The notebook has no automated tests, dependency manifest, packaging, deployment configuration, or persistent service process.
- HTML is stripped with a regular expression, which is not a robust HTML-to-text parser.
- Broad exception handlers can hide parsing and mailbox errors, and failures default to ham.
- The reported confidence is the maximum probability for the predicted class, not specifically the probability of spam.
- Copy/delete/expunge behavior depends on server mailbox naming and can permanently change the inbox.
- Pickle loading is unsafe for untrusted artifacts and may fail across scikit-learn versions.

## Resume Evaluation Notes

- **Strongest technical aspects:** End-to-end integration of email retrieval, MIME parsing, text preprocessing, TF-IDF inference, mailbox actions, model persistence, and scheduling.
- **Most impressive implementation:** The IMAP processing pipeline that decodes messages, classifies their content, discovers a spam mailbox, and moves predicted spam.
- **Weak/basic parts:** Very small synthetic training set, training-only evaluation, regex-based HTML handling, broad exception handling, notebook-only execution, and no tests or dependency file.
- **Verified metrics:** 1.00 in-sample accuracy on 24 training examples; five saved demo predictions with displayed confidence between 0.59 and 0.63.
- **What is unique:** It combines a simple NLP classifier with direct Gmail IMAP mailbox automation instead of stopping at offline text classification.
- **What may be worth highlighting on a resume:** Building the end-to-end Python/IMAP/NLP workflow, multipart email extraction, persisted TF-IDF and logistic regression inference, and scheduled mailbox processing, described explicitly as a prototype.
- **Claims that cannot be verified from the repo:** Real-world accuracy, precision/recall on unseen email, number of inbox messages successfully processed, users, deployment uptime, performance, scale, or measurable impact.
