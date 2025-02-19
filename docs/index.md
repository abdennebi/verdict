---
label: "Quickstart"
icon: zap
order: 9
---

# Verdict
!!!primary Tip
Jump right into our [Concepts Overview](./concept/overview.md) for a birds-eye view of Verdict execution flow.
!!!

Verdict is a declarative framework for specifying and executing compound LLM-as-a-judge systems. It is
- Plug-and-play across providers/models, prompts, extraction methods, and judge protocols allowing for rapid iteration and tweaking.
- Arbitrarily composable, with dependency graph primitives found across the LLM-as-a-judge research literature.
- Massively concurrent out-of-the-box and can easily coordinate thousands of LLM calls and responses simultaneously.
- Rate-limited on the client-side, so you never lose experiment results halfway through.
- Integrated with DSPy for use as a metric in AI system optimization; and more to come soon!
- SOTA-level on a variety of benchmarks with minimal fitting and a fraction of the inference time.

## Quickstart
Verdict allows you to define arbitrary compound LLM-as-a-judge systems in a declarative manner. Here, we show an example of a Verdict judging pipeline that enables us to achieve SOTA  **(+14.5% over GPT-4o)** on the ExpertQA hallucination detection dataset.

First, install the package with `pip install verdict` (or with `uv`).

We'll define a judging protocol that consists of
1. A GPT-4o judge who initially determines whether a hallucination is present
2. A second verification GPT-4o judge who reviews and validates the first judge's reasoning and decision

We then repeat this process 3 times, and aggregate the resulting validated scores using a majority vote.

### Verdict Pipelines & Prompts
Let's sketch this idea out using the Verdict DSL. We'll define a `judge_then_verify` protocol, layer 3 of them into an ensemble using a `Layer`, and aggregate the results with a `MaxPoolUnit`. Using the `>>` operator to define a dependency between LLM calls, we can compose arbitrarily complex judge protocols as shown below.

```python
from verdict import Pipeline, Layer
from verdict.common.judge import CategoricalJudgeUnit
from verdict.scale import DiscreteScale
from verdict.transform import MaxPoolUnit

#   1. A CategoricalJudgeUnit first decides if a hallucination is present...
#   2. Followed by a CategoricalJudgeUnit that verifies the initial judge's explanation and decision.

judge_then_verify = CategoricalJudgeUnit(name='Judge', categories=DiscreteScale(['yes', 'no']), explanation=True).prompt("""
    Determine whether the provided claim is consistent with the corresponding document. Consistency in this context implies that
    all information presented in the claim is substantiated by the document. If not, it should be considered inconsistent.

    Document: {source.doc}
    Claim: {source.claim}

    Please assess the claim’s consistency with the document by responding with either "yes" or "no".
""").via(policy_or_name='gpt-4o', retries=3, temperature=0.7) \
>> CategoricalJudgeUnit(name='Verifier', categories=DiscreteScale(['yes', 'no'])).prompt("""
    Check if the given answer correctly reflects whether the claim is consistent with the corresponding document.

    Document: {source.doc}
    Claim: {source.claim}

    Answer: {previous.choice}
    Answer Justification: {previous.explanation}

    Respond "yes" if the claim is consistent with the document and "no" if the claim is not consistent with the document.
    """).via(policy_or_name='gpt-4o', retries=3, temperature=0.0)

# ...and then aggregate the results of the three judges+verifiers with a `MaxPoolUnit`
pipeline = Pipeline('HierarchicalVerifier') \
    >> Layer(judge_then_verify, repeat=3) \
    >> MaxPoolUnit()

# Graphical representation of the Verdict system
pipeline.plot() # requires Graphviz binaries
```

![](/static/quickstart/pipeline.png)

That's a lot! Let's break it down.

#### Units, or LLM Calls
A `Unit` is the simplest building block of a Verdict pipeline. It represents a single LLM call, and is responsible for taking in some well-defined input, performing model inference, and returning some well-defined output. We provide a number of [built-in units](./concept/unit.md#built-ins); here we use a `CategoricalJudgeUnit` to select some discrete output from a set of categories.

#### Prompts
How can we access the outputs of the first `CategoricalJudgeUnit` within the second `CategoricalJudgeUnit`? This is where the `previous` keyword comes in handy -- it points to the output of the previous `Unit` in the pipeline. If there are multiple previous units, we provide simple ways to [disambiguate between them](./concept/prompt.md#previous). We can also inject inputs from the source dataset using `{source.doc}`.

#### Model Inference Parameters
Each `Unit` can specify its inference model as well as number of retries and inference parameters like `temperature`. We provide a ton of [flexibility here](./concept/model/model.md) for configuring provider or self-hosted models.

#### Layering It All Together
We can use a `Layer` to layer arbitrary subgraphs together. We make it [easy to customize](./concept/layer.md) how the subgraphs are stitched together and to downstream units. For example, we can broadcast units in a 1-1 fashion, stitch them together in a round-robin configuration, or have them fully-connected.

#### Aggregation
We aggregate the results of the three judges+verifiers with a built-in `MaxPoolUnit`. This is a simple [transformation](./concept/transform.md) that performs a majority vote. Verdict allows you to place arbitrary logic at any stage of the pipeline using a `MapUnit`.

### Execution
We can run this pipeline over a test sample. Verdict will send out each LLM call in its own worker thread as soon as all dependencies are met.
```python
from verdict.schema import Schema

test_sample = Schema.of(
    doc="by the mid-2010s, streaming media services were...",
    claim="One of the key initiatives in this regard was..."
)

# Don't forget to set your API key!
# $ export OPENAI_API_KEY=*************************
response, _ = pipeline.run(test_sample, max_workers=64)

print(response['HierarchicalVerifier_root.block.block.unit[Map MaxPool]_choice'])
# 'no'
```

### Datasets
We'll pull in the `LLM-AggreFact` dataset from HuggingFace, filter it to only include the `ExpertQA` dataset, and prepare it for a Verdict pipeline.
```python
from datasets import load_dataset
from verdict.dataset import DatasetWrapper

# We'll disable Verdict rate-limiting for this example. By default, Verdict follows the OpenAI Tier 1 rate limits for `gpt-4o-mini`.
from verdict.util import ratelimit
ratelimit.disable()

# Load the ExpertQA dataset (https://arxiv.org/pdf/2309.07852)
dataset = DatasetWrapper.from_hf(
    load_dataset('lytang/LLM-AggreFact').filter(
        lambda row: row['dataset'] == 'ExpertQA'
    ),
    columns=['claim', 'doc', 'label'], # we can also specify a custom input_fn to map each row into a Schema
    max_samples=10 # randomly sample 10 examples for demo purposes
)

# all responses from intermediate Units are available as columns in response_df!
response_df, _ = pipeline.run_from_dataset(dataset['test'], max_workers=512)
response_df['pred_label'] = response_df['HierarchicalVerifier_root.block.block.unit[Map MaxPool]_choice'] == 'yes'

from verdict.util.experiment import display_stats, ExperimentConfig
display_stats(response_df, ExperimentConfig(ground_truth_cols=['label'], prediction_cols=['pred_label']));
```

---

### Putting it All Together
View [this notebook](https://colab.research.google.com/github/haizelabs/verdict/blob/main/notebooks/results/hierarchical.ipynb) for a complete example (with prompts) that reproduces our SOTA results on the ExpertQA hallucination detection dataset.

![](/static/quickstart/results.png)
