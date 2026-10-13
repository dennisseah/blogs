# Decision Models

TypeSafe AI recently released Jev, a specialized AI model for making fast,
structured decisions in software. Given text or JSON data and predefined
questions, Jev returns choices, scores, and probability estimates that
applications can use for classification, routing, triage, and similar workflows.
Early reports suggest that it can reduce latency and cost compared with using
large language models for these narrow tasks. However, Jev does not collect or
analyze real-time data on its own, and it is not a general predictive-modeling
system. Its architecture has not been fully disclosed, and its broader accuracy
claims have not yet been independently established.

## Microsoft-Decision-1

Microsoft has also introduced
[Microsoft-Decision-1](https://commandline.microsoft.com/microsoft-decision-1-model-foundry/),
a decision-scoring model available through Microsoft Foundry and OpenRouter.
Built by post-training Qwen3.5-9B, it evaluates text or JSON and returns
probabilities for fixed yes-or-no, multiple-choice, and rating options instead
of generating prose. Microsoft positions the model for classification, routing,
prioritization, verification, grading, and workflow control. Although Microsoft
has identified the base model and its intended interface, it has not published
the derived model weights, detailed training data and methods, architectural
modifications, or enough implementation information to reproduce the model. In
internal tests covering 36 benchmarks and nearly 150,000 questions, Microsoft
reported that it achieved the highest accuracy and lowest latency among the
models evaluated. These results are Microsoft-reported and cannot yet be fully
reproduced independently, so teams should follow the
[Foundry guidance](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/how-to/use-foundry-models-microsoft-decision)
and validate its accuracy, thresholds, and latency against their own labeled
data before using it to automate decisions.

## Comparison and Use Cases

Both Jev and Microsoft-Decision-1 are best suited to narrow, low-risk tasks in
which an incorrect result can be detected and reversed, such as routing support
tickets or prioritizing a work queue. A structured response guarantees a valid
output format, not a correct judgment, so applications should validate the
models against representative labeled data, define confidence thresholds, record
decisions for auditing, and send uncertain or consequential cases to a person.
Deterministic code should continue to handle arithmetic, dates, permissions, and
mandatory policies. Neither model should act as the sole decision-maker for
medical, legal, employment, credit, security, or other high-impact decisions
without domain-specific safeguards, ongoing monitoring, and meaningful human
oversight.

## Pricing

The current published price for both Jev and Microsoft-Decision-1 is $0.042 per
million input tokens, with no charge for output tokens. This input-only model
can make high-volume classification, routing, and scoring substantially less
expensive than using a generative model that bills for both the prompt and a
potentially long response. Decision models also return compact probabilities and
labels instead of generating explanatory prose, which keeps output predictable
and avoids paying for unnecessary text. Costs can still grow when applications
repeatedly send large states, retry requests, or evaluate many questions, so
teams should minimize irrelevant input, monitor actual token usage, and confirm
current provider pricing before estimating production expenses.

## Demo

I have created a demo to illustrate how to use the decision models for routing
customer requests. The demo includes a sample state, criteria for different
departments, and a Python script that sends a request to the
Microsoft-Decision-1 model and prints the structured response. Here is the
Python script for the demo:

> [!NOTE]
> At the time of writing, a dedicated Python SDK for the
> Microsoft-Decision-1 decision endpoint is not available. The demo therefore
> uses `requests` to call the Foundry REST endpoint directly while
> `DefaultAzureCredential` provides Microsoft Entra ID authentication.

```python
import json
import os

import requests
from azure.identity import DefaultAzureCredential
from dotenv import load_dotenv

load_dotenv()

state = "Customer needs assistance with canceling an appointment"

criteria = {
    "appointment": "services involve making reservations and making changes to reservations which include canceling, rescheduling, and confirming appointments.",
    "ordering": "ordering of foods, drinks, and other items from restaurants or stores, including delivery and takeout options.",
    "customer_service": "services that assist customers with inquiries, complaints, and support related to products or services.",
    "product_inquiry": "services that provide information about products, including specifications, availability, and pricing.",
    "technical_support": "services that assist customers with technical issues related to products or services, including troubleshooting and problem resolution.",
    "feedback": "services that collect customer feedback, including surveys, reviews, and suggestions for improvement."
}

endpoint = os.environ.get("FOUNDRY_ENDPOINT")
deployment_name = os.environ.get("FOUNDRY_DEPLOYMENT_NAME")

credential = DefaultAzureCredential()
access_token = credential.get_token(
    "https://cognitiveservices.azure.com/.default"
).token

headers = {
    "Authorization": f"Bearer {access_token}",
    "Content-Type": "application/json",
}

payload = {
    "model": deployment_name,
    "state": state,
    "questions": {
        "department": {
            "type": "choice",
            "instructions": (
                "Which department should handle this customer request?"
            ),
            "criteria": criteria,
        }
    },
}

response = requests.post(
    endpoint,
    json=payload,
    headers=headers,
    timeout=60,
)
if not response.ok:
    try:
        error_details = json.dumps(response.json(), indent=2)
    except requests.exceptions.JSONDecodeError:
        error_details = response.text
    raise RuntimeError(
        f"Foundry request failed with HTTP {response.status_code}: "
        f"{error_details}"
    )
print(json.dumps(response.json(), indent=2))
```


the output of the script will show the structured response from the
Microsoft-Decision-1 model, indicating which department should handle the
customer request based on the provided state and criteria.

```json
{
  "model": "microsoft-decision-1",
  "answers": {
    "department": {
      "choice": "appointment",
      "confidence": 0.9943241149259044,
      "probabilities": {
        "appointment": 0.9952700957715871,
        "customer_service": 0.0008009267414559044,
        "feedback": 0.0010284102929342725,
        "ordering": 0.0011653415325564727,
        "product_inquiry": 0.0007068153685320087,
        "technical_support": 0.0010284102929342725
      },
      "type": "choice"
    }
  },
  "usage": {
    "input_tokens": 160,
    "output_tokens": 1
  }
}
```

The JSON response identifies the underlying model and groups each result under
the corresponding question name, which is `department` in this example. The
`choice` field contains the option selected by the model, while `probabilities`
shows its estimated probability for every available option. The `confidence`
value summarizes how strongly the distribution favors the selected choice, but
it does not guarantee that the decision is correct. Applications can compare
these values with tested thresholds and send uncertain results for human
review. The `usage` object reports the tokens processed by the request; this
example used 160 input tokens and one output token.

## Conclusion

Decision models offer a promising alternative to general-purpose language
models when an application needs fast, inexpensive, and consistently structured
classification or scoring. Their compact responses can simplify software
integration, reduce latency, and lower token costs for high-volume workflows.
However, Jev and Microsoft-Decision-1 are still new, their internal designs are
not fully disclosed, and independent evidence about their long-term accuracy,
reliability, and behavior remains limited. Enterprises can begin with controlled
pilots for low-risk, reversible tasks while measuring results against labeled
data and maintaining human review. Broader adoption, especially for critical
business decisions, should wait until the models have a longer production
record, stronger independent validation, mature governance practices, and clear
operational support from providers.
