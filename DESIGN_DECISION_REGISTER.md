# Design Decision Register

> **_Intended audience:_** _Engineers, TPMs, Head of Engineering_
> 
> **_Purpose:_** _A live document which collates design decisions & analysis for sharing across projects_

## Resources



> [!NOTE] 
> **Below are some of the key classes and functions that you will need to understand in order to build an evaluation in Inspect. The accompanying code and documentation (if defined) is hyperlinked for each component.**
>*   **Task:** [code](https://github.com/UKGovernmentBEIS/inspect_ai/blob/main/src/inspect_ai/_eval/task/task.py)
>*   **TaskState:** [code](https://github.com/UKGovernmentBEIS/inspect_ai/blob/main/src/inspect_ai/solver/_task_state.py)
>*   **Dataset/Sample**: [documentation](https://inspect.ai-safety-institute.org.uk/datasets.html) | [code](https://github.com/UKGovernmentBEIS/inspect_ai/blob/main/src/inspect_ai/dataset/_dataset.py)
>*   **Sandbox:** [documentation](https://inspect.ai-safety-institute.org.uk/sandboxing.html) | [code](https://github.com/UKGovernmentBEIS/inspect_ai/tree/main/src/inspect_ai/util/_sandbox)
>*   **Solver:** [documentation](https://inspect.ai-safety-institute.org.uk/solvers.html) | [code](https://github.com/UKGovernmentBEIS/inspect_ai/blob/main/src/inspect_ai/solver/_solver.py)
>*   **Basic Agent:** [documentation](https://inspect.ai-safety-institute.org.uk/agents.html) | [code](https://github.com/UKGovernmentBEIS/inspect_ai/blob/main/src/inspect_ai/solver/_basic_agent.py)
>*   **Scorer:** [documentation](https://inspect.ai-safety-institute.org.uk/scorers.html) | [code](https://github.com/UKGovernmentBEIS/inspect_ai/blob/main/src/inspect_ai/scorer/_scorer.py)
>   *   **Score:** [code](https://github.com/UKGovernmentBEIS/inspect_ai/blob/main/src/inspect_ai/scorer/_metric.py)
>    *   **Metric:** [code](https://github.com/UKGovernmentBEIS/inspect_ai/blob/main/src/inspect_ai/scorer/_metric.py) | [documentation](https://inspect.ai-safety-institute.org.uk/scorers.html#scoring-metrics)


# Agent Design

## **There's a particular step or action that is causing the agent to perform badly on a (sub)task - should I build in additional affordances (tools) to help improve performance?**

Before adding in additional affordances, you should consider the following:

* Have you correctly identified the root cause of the bottleneck? Are there other parts of the evaluation that could be causing the agent to fail (i.e., environment is not set up properly, the message/token limits are too low, the task is not explained properly, the task is too complex and needs to be broken down into subtasks)?
* How does the additional affordance impact task difficulty? Does the tool trivialise the core capability being tested? (E.g. you are evaluating the coding abilities of an agent and prewrite Python scripts that fundamentally solve the coding challenge being tested)

## **Can we use the basic agent, or do we need to use custom scaffolding?**

By default, you **must** always try and leverage the basic agent implemented in Inspect before considering custom agent scaffolding. We want to use the basic agent if it is sensible to do so as this will:

* Give more time to spend on other aspects of the evaluation.
* Reduce overhead for testing and validation of custom agent scaffolding.
* Reduce maintenance overhead for AISI.

There are a few tricks/hacks when it comes to using the basic agent that may be not obvious at first glance. If you are unsure how the basic agent works, review the following resources:

* The basic agent [documentation](https://inspect.ai-safety-institute.org.uk/agents.html#sec-basic-agent).
* The `basic_agent()` code [implementation](https://github.com/UKGovernmentBEIS/inspect_ai/blob/main/src/inspect_ai/solver/_basic_agent.py).
* The basic agent loop flow chart:

  ```mermaid
    flowchart TD
        subgraph Loop["Basic Agent Loop"]
            direction LR
            Generate["Generate"] --> ModelTool
            ModelTool{"Tool<br>called?"}
            ModelTool -->|Yes| SubmitTool{"Submit tool<br>called?"}
            ModelTool -->|No| AppendContinue["Append continue<br>message"]
            
            AppendContinue --> LimitCheck
            SubmitTool -->|No| LimitCheck{"Limit<br>reached?"}
            SubmitTool -->|Yes| MaxAttempts{"Max<br>attempts?"}
            
            MaxAttempts -->|No| Score["Score<br>answer"]
            Score --> Correct{"Agent answer<br>correct?"}
            
            Correct -->|No| AppendIncorrect["Append incorrect<br>message"]
            AppendIncorrect --> LimitCheck
            
            LimitCheck -->|No| Generate
        end
        
        Start["Initialise system message, available tools, chat message/token limits, and max attempts"] --> Generate
        MaxAttempts -->|Yes| Exit["Exit"]
        Correct -->|Yes| Exit
        LimitCheck -->|Yes| Exit

        classDef condition fill:#be133a,stroke:#8e0e2b
        class MaxAttempts,Correct,LimitCheck,Exit condition
  ```

Reasons why you may not be able to leverage the basic agent include (though these are typically out of scope):

* **You** **need** **to add custom components** (solvers) to enhance the agent scaffolding (e.g. a Chain of Thought reasoning solver).
* **You need to add another exit condition** **to the agent loop** or need to remove/modify one of the existing exit conditions. Currently the basic agent loop is terminated if one of the following occurs:
  * The agent has reached the maximum submission attempts.
  * The token or message limit has been reached.
  * The agent agent's submitted answer is correct.

## The submission logic on the basic agent requires the agent to submit an answer, but this doesn't make sense for my evaluation.

_(For richer context) The basic agent submission functionality is designed to accept an "answer" from an agent which is evaluated for correctness. When the agent calls `submit`, the agent is scored using the task's configured `scorer`. A score of 1.0 (or a value that converts to 1.0 via the `score_value` function) indicates success, ending the agent loop. Otherwise, the agent receives an "incorrect" message prompting it to try again._

This submission logic can be confusing when your task doesn't involve finding a specific answer, but rather evaluating the agent's process or output (e.g. code changes, design proposals, etc). While this submit-and-score flow is hardcoded into the basic agent, you can adapt it via:

* `submit_description`: Lets you customise how the submit tool is described to the agent (e.g., you can change it from "Submit an answer for evaluation" to \`"Use 'submit' to submit your design")
* `incorrect_message`: Specifies what feedback the agent receives after an incorrect submission (e.g. from "The submitted answer was incorrect" to "The submission was incorrect")

Further, the scoring mechanism is flexible. `score(state)` evaluates based on the task's `scorer` , which doesn't have to use the answer submitted by the agent. As long as your `scorer` returns 1.0 for success (or your `score_value` function maps successful scores to 1.0), the submission flow works.


```python
INCORRECT_MESSAGE = "Your submission failed unit tests. Please revise your changes and try again."
SUBMIT_DESCRIPTION = "Use the word 'submit' to submit your code changes."

def default_solver() -> Solver:
    return basic_agent(
        submit_description=SUBMIT_DESCRIPTION,
        incorrect_message=INCORRECT_MESSAGE 
    )
```

# Datasets & Resources

## Should I use an external dependency or (if possible) should I make this a persistent resource and save it in the evaluation module?

Per the [AS Evaluation Standard](https://ukgovernmentbeis.github.io/as-evaluation-standard/) under Setup and Dependencies, the evaluation should require minimal extra dependencies beyond those already in the template repo. If an external dependency is used, you should consider why the dependency is necessary and assess any potential risks of abandonment/ dependency decay.

Some reasons for external dependencies might be:

* The dataset is too large
* The dependency provides complicated functionality that would be impractical to reimplement
* The license doesn't support bundling
* You are confident that the resource won't be depreciated

Some reasons you may want to make a dependency persistent include:

* The resource changes over time (evaluations should be consistent over time so that the results can be comparable)
* External service reliability could impact evaluation results
* The dependency is simple enough to reimplement or mock
* The risk the dependency will be abandoned or unsupported is too high

If your evaluation has any external dependencies, the versions of those dependencies **must** be pinned.

## Should I use Inspect `Dataset` readers to handle datasets that will be used in the evaluation, but wont be used as the `Dataset` to run the `Task` **and** evaluate the model?

You should only use the Inspect `Dataset` readers (such as `csv_dataset`, `json_dataset`, `hf_dataset`, etc) for loading datasets that will be used _as your evaluation dataset._ More clearly, `Dataset` objects are designed for running an evaluation on a model and should not be used to load any dataset besides the one that you will pass as the `dataset` on your `Task`.

There are a few reasons for this:

1. The readers enforce a specific structure (requiring input, and other optional fields)
2. They create `Sample` objects with special fields used by Inspect's evaluation pipeline
3. Standard Python libraries provide more flexibility in how you structure and process the data
4. You avoid potential confusion between evaluation datasets and auxiliary data

Filtering, shuffling, and slicing functionalities provided by Inspect's `Dataset` class are also readily available in other Python libraries (such as `datasets`, `pandas`, and `numpy`).

## How do I setup my `Dataset` if the evaluation only has one complex task instead of multiple samples to evaluate?

For agentic evaluations, the term "dataset" may seem misleading since you might only have one complex task rather than multiple examples to evaluate. In these cases, your `Dataset` should consist of just a single `Sample` where the `input` describes the task objectives and success criteria. You should then read this sample using `MemoryDataset` and pass this to the `Task`. See a very simple mocked example below:

```python
PROMPT = "INSERT PROMPT HERE"

def _get_dataset() -> Dataset:
    # Create a single sample for the evaluation
    eval_sample = Sample(
        input=PROMPT,
        # Include any other relevant parameters here, such as `files`, `metadata`, etc
    )
    # Return memory dataset
    return MemoryDataset([eval_sample])

def evaluation_task():
  return Task(
    dataset = _get_dataset()
    # Also add scorer, solver, sandbox, etc
)


```

# Scoring

## How do I turn my continuous score into a binary (pass/fail) metric?

There's no science to this, but here are some guiding principles:

* Getting a passing grade should be **hard (for a current LLM)**, but **doable (for a human)**. Thus, human performance should provide a rough upper-bound to a threshold threshold, and sub-frontier model performance should provide a rough lower-bound.
* Consider phrasing your evaluation as a sentence: "if a model can pass this evaluation, then it can <do some behaviour>". Try to make the behaviour as real-world as possible: "do the job of a research assistant" is better than "perform data retrieval, paper summarization, and data preprocessing tasks upon prompting". Then ask yourself: at what threshold would a reasonable, well-informed person agree that this model satisfies that description?
* If your evaluation is specifically measuring a dangerous capability, or a proxy thereof, choose a threshold such that passing the evaluation means the agent more likely than not possesses this capability.

## I want to collect scores throughout the evaluation run rather than just scoring some final agent output or answer. How can I do this?

You can capture intermediate scores throughout the evaluation by using a `Store`. The `TaskState` for each sample has a `Store` that can be used to collect any data throughout the evaluation. The `Store` an be accessed directly from the `TaskState` via `state.store` or can be accessed using the `store()` global function.

When a tool or solver runs, you can store any captured data you want to score using the `Store`'s `get` and `set` functions. Then, when the scorer is called at the end of the evaluation run, you can use `get` to retrieve the stored data and use the data to score the agent. See mocked example below:

```python
from inspect_ai.util import store

@tool()
def some_tool() -> Tool:
    async def execute() -> str:
        # Get list of previously stored results
        results = store().get("results")

        # Get some result
        result = await some_function()

        # Append new result 
        results.append(result)
        
        # Store results
        store().set("result", result)

        return "Hooray"
    return execute

@scorer(metrics=[])
def scorer():
   async def score(state: TaskState, target: Target) -> Score:
      # Get result
      results = state.store().get("results")

      # Now use results to score agent...
```

## How can I define metrics when I only have one sample?

`Metric` 's are typically calculated by aggregating `Score` values across multiple `Sample` 's into statistical measures, such as accuracy, mean, standard error, and other quantitative indicators of model performance. This can seem confusing or arbitrary when you only have one `Sample`. However, even if your dataset consists of only a single `Sample`, there is still benefit to defining metrics as these display in the top right corner of Inspect View and give richer context for your evaluation

**Inspect View Example**

<div align="center">
  <img src="https://t9011772876.p.clickup-attachments.com/t9011772876/0aba2379-01c9-4a41-afb0-f9d4d8f69b87/image.png" width="700" height="auto">
</div>

For displaying metrics when you only have one `Sample`, you should calculate any metrics inside the `scorer` and then retrieve the metrics inside `@metric` decorated functions in one of two ways:

1. Via the `score.value`. A `score.value` can be a dictionary, which means you can set a key/value pair for each of your metric values and retrieve them in the `@metric` function.
2. Via the `score.metadata`. Create a key/value pair for each of your metric values in the score metadata. _This method is preferred if there is a single value that would make sense for your `score.value`, since other parts of Inspect work best when a score has a single value._

```python
@metric
def metric_1() -> Metric:
    def metric(scores: list[Score]) -> float:
        # Method 1: Using the value
        return scores[-1].value["metric1"]  # type: ignore

    return metric

@metric
def metric_2() -> Metric:
    def metric(scores: list[Score]) -> float:
        # Method 2: Using the metadata
        return scores[-1].metadata["metric_2"]  # type: ignore

    return metric


@scorer(metrics=[])
def scoring_function():
    async def score(state: TaskState, target: Target) -> Score:

        metric_1 = 0.5
        metric_2 = 0.4

        return Score(
            value={
                "metric_1": metric_1,
            },
            metadata={
                "metric_2": metric_2
            }
        )
    return score
```

For these metrics to make sense, your dataset must have only one sample. Thus, it is recommended to write a unit test that verifies your task only has a single sample:

```python
def test_task_has_only_one_sample():
    assert len(my_task().samples) == 1
```

## What scores should I report for a "baseline improvement" evaluation?

For some evaluations, we have some baseline performance on a task, and we're asking an agent to improve this performance. An example would be "elicitation" evaluations: given some benchmark, and the score obtained using basic scaffolding, can the agent modify the scaffolding so that a "child model" using that scaffolding gets a better score?

If your evaluation fits this description, you should report 3 scores:

* An absolute score, which represents the task performance after the agent made its modifications;
* A relative score, calculated by normalizing the absolute score by the baseline;
* A binary uplift score, calculated by applying a threshold to the relative score.

For advice on choosing a suitable threshold, refer to [scoring guidelines](#how-do-i-turn-my-continuous-score-into-a-binary-passfail-metric).

These three scores can be reported by using [dict-valued Score objects](https://inspect.ai-safety-institute.org.uk/scorers.html#value).

# Sandbox

## How should I configure my environment variables?

You **should** put any environment variables needed in a `.env` file, inside the inner folder of your evaluation. For example, if your evaluation is called `my_eval` , you should place any environment variables in `my_eval/my_eval/.env` .

This is because, for some purposes (e.g. using build args in docker compose), only this directory is searched when looking for `.env` files.

Tip: since Inspect searches in the current working directory and parent directories for the `.env` file, you'll need to ensure your current working directory is `my_eval/my_eval` when you run the eval. If you want to run your eval from any directory, a workaround is to add the following block to the end of your `my_eval.py` file:

```python
if __name__ == "__main__":
    from contextlib import chdir
    from inspect_ai import eval
    from pathlib import Path
    
    with chdir(Path(__file__).parent):
        eval(my_task())
```

Just make sure to remove this block before submitting the code to AISI.

You **must** document any required environment variables in your [README.md](http://README.md).

## `sandbox().exec()` is returning an empty `ExecResult`. What should I do?

Sometimes, strange things will happen when running commands inside the docker sandbox while inside a dev container. Concretely, suppose you are running Inspect from inside a Dev Container, and you've configured your task to use a Docker-based sandbox:

```python
@task
def my_task() -> Task:
    return Task(
        ...,
        sandbox=("docker", str(Path(__file__).parent / "compose.yaml")),
    )
```

Sometimes, when executing commands inside the sandbox, you might get strange results:

```python
sandbox().exec(["python", "script.py"]) # returns empty ExecResult
```

If this happens, try running the same code but outside of the Dev Container.
