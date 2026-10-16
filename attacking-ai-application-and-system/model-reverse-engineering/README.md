# Model Reverse Engineering

This lab approximates a query-accessible penguin classifier using synthetic inputs and its returned class labels. The target accepts flipper length and body mass and predicts Adélie or Gentoo. Its model parameters and original training data are not required for the surrogate training procedure.

## Working procedure

Set `CLASSIFIER_URL` to the root URL of the current authorized lab instance. The instance address is temporary. The run documented here used 1,000 uniformly sampled feature pairs, a standardized logistic-regression pipeline, and a one-dimensional label array.

```python
import random

import joblib
import pandas as pd
import requests
from sklearn.linear_model import LogisticRegression
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import StandardScaler

CLASSIFIER_URL = "http://TARGET/"
N_SAMPLES = 1000

samples = {"Flipper Length (mm)": [], "Body Mass (g)": []}
for _ in range(N_SAMPLES):
    samples["Flipper Length (mm)"].append(random.uniform(150, 250))
    samples["Body Mass (g)"].append(random.uniform(2500, 6500))

samples_df = pd.DataFrame(samples)

predictions = {"species": []}
for i in range(N_SAMPLES):
    sample = {
        "flipper_length": samples["Flipper Length (mm)"][i],
        "body_mass": samples["Body Mass (g)"][i],
    }
    response = requests.get(CLASSIFIER_URL, params=sample, timeout=10)
    response.raise_for_status()
    prediction = response.json().get("result")
    if prediction is None:
        raise ValueError(f"No class label returned for query {i}")
    predictions["species"].append(prediction)

predictions_df = pd.DataFrame(predictions)
surrogate_model = make_pipeline(StandardScaler(), LogisticRegression())
surrogate_model.fit(samples_df, predictions_df.values.ravel())
joblib.dump(surrogate_model, "student.joblib")

with open("student.joblib", "rb") as model_file:
    response = requests.post(
        CLASSIFIER_URL + "model",
        files={"file": ("student.joblib", model_file)},
        timeout=30,
    )
response.raise_for_status()
evaluation = response.json()
print({
    "accuracy": evaluation.get("accuracy"),
    "accepted": bool(evaluation.get("flag")),
})
```

## Observed result

The successful run returned an evaluator accuracy of **0.9817518248175182 (98.18%)** and an acceptance token. The token is omitted from this public writeup. The score is a single external evaluation of one unseeded query set; later runs can differ. It establishes predictive agreement on the evaluator's test set without establishing recovery of the original model parameters.

The [module notebook](https://github.com/skilfoy/htb-ai-red-teamer/tree/main/attacking-ai-application-and-system) will develop this experiment within a broader analysis of application and system attack surfaces.
