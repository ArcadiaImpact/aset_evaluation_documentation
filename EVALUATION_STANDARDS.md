# ASET Evaluation Standards

## General Evaluation Standards

These guidelines apply to the design and implementation of any LLM evaluation, independent of the framework used.

*   **Evaluation Design**
    
    *   **Define Goals Clearly:**
        *   Identify the capability or behavior being evaluated.
        *   Focus on meaningful capabilities, avoiding superficial or overly broad metrics.
    *   **Set Pass/Fail Criteria:**
        *   Establish clear, objective metrics for success and failure. A reasonable, well-informed person should conclude that a model meeting the success criteria exhibits the behaviour being evaluated.
            *   These pass/fail criteria should be on the individual sample level, not the evaluation as a whole.
        *   Incorporate both positive and negative test cases.
    *   **Handle Stochasticity:**
        *   Design evaluations to accommodate variability in LLM outputs (e.g., using confidence intervals or multiple runs).
    
*   **Prompt Engineering**
    
    *   **Consistency:** Maintain uniform system prompts across tasks.
    *   **Avoid Leakage:** Ensure prompts do not inadvertently reveal correct answers.
    *   **Clarity:** Use explicit, unambiguous instructions.
    *   **Diversity in Few-shot Examples:** Ensure examples represent a wide range of scenarios.
    *   **Avoid Overfitting:** Split dataset into dev & validation splits. Only use the dev split for prompt engineering; use the validation split sparingly
    *   **Optimal:** Ensure that prompts are _locally_ optimal by testing some sensible perturbations to verify that no small changes significantly enhance task performance. Optimality cannot be guaranteed, but engineers should run evaluations with these variants or interact with the model to confirm that the prompt is robust and effectively understood by the model. Some example prompt perturbations are:
        *   **Formatting Changes:** Switch between bullet points and numbered lists.
        *   **Paragraph Reordering:** Change the sequence of informational sections.
        *   **Instruction Variations:** Rephrase instructions for clarity or emphasis.
        *   **Question Modifications:** Alter the way questions are posed to the model.
        *   **Detail Adjustments:** Add or remove specific details to see their impact on responses.
    
*   **Dataset Management**
    
    *   **Data Quality:** Ensure datasets are clean, relevant, and free from errors or biases.
    *   **Separation of Data:** Maintain a strict boundary between training and evaluation datasets.
    *   **Document Processing:** Record preprocessing steps, filters, and validation methods.
    *   **Dataset Size**: Ensure the dataset is large enough to detect differences between models most of the time. In practice, that means a dataset should have ≥1000 records, given reasonable assumptions. If you think your dataset doesn't need that many records, you should write down your assumptions and perform a power analysis as per section 5 of [Miller, 2024](https://arxiv.org/pdf/2411.00640).
    
*   **Implementation Standards**
    
    *   **Reproducibility:** Use deterministic methods where possible to ensure consistent results.
        *   Concretely, this means whenever pseudo- or real randomness is used, explicitly or implicitly, the random generators should be seeded beforehand.
        *   This reproducibility should be witnessed by unit tests.
    *   **Error Handling:** Implement robust logging, timeouts, and retry policies.
    *   **Version Control:** Track all changes to code, datasets, and configurations.
    *   **Efficiency:** Ensure code runs quickly and uses minimal resources.
        *   Run performance tests (at least informally) and report the results in the README.
        *   For any processing that is common across multiple runs, perform the computation ahead of time and cache the results.
        *   Prioritise performance and resource-efficiency when choosing between external tools or services. For example, in most cases using a model API will be preferred over running a local model (resource-intensive and often slower), unless access to the model weights is required or the environment doesn't have access to the internet.
    *   **Maintenance:** Ensure code is as simple as possible. Prefer simple-but-long over complex-but-short.
    
*   **Security Considerations**
    
    *   **Sanitization:** Ensure inputs and outputs are safe from injection attacks or unexpected behavior.
    *   **Rate Limiting:** Implement resource constraints and timeouts to prevent abuse.
    *   **Sandboxing:** Use secure environments for code execution tasks.
        *   If an agent needs access to a sub-model, do not give unrestricted access to an API. Instead, give the agent controlled access via one of:
            *   role-based access (ideally with dynamically generated credentials).
            *   locally deployed sub-models.
            *   custom tools.
        *   Protect against abuse of inference APIs:
            *  use a read-only API key.
            *  set a budget limit (i.e. if there's a risk of a model abusing the API, create a new OpenAI project for that eval specifically and set budget alerts and limits).
    *   **Secrets:** Ensure secrets are not committed to code repos, available to agents, or available to AI code tools such as Copilot or Cursor.
        *   Secrets files (\`.env\` or similar) should be added to `.gitignore` .
        *   A secret scanner (e.g. [Github Secret Scanner](https://docs.github.com/en/code-security/secret-scanning/introduction/about-secret-scanning)) should not find any secrets in the codebase.
    
*   **Scoring and Metrics**
    
    *   **Automated Scoring:** Design evaluations that can be scored without manual intervention.
    *   **Binary Metrics:** Include simple pass/fail metrics, supplemented by richer auxiliary metrics:
        *   e.g. if there are multiple (finite, easily enumerable) ways to complete a task, tracking that in a metric might be valuable.
        *   Utilize built-in tracking by the evaluation framework for managing non-task-specific metrics (e.g., token count, tool calls, total API cost). Inspect and Vivaria already do this so eval developers do no need to implement these metrics manually.
    *   **Edge Cases:** Document and test handling of unusual or extreme inputs.
    
*   **Documentation and QA**
    
    *   **Comprehensive Documentation:** Record assumptions, limitations, and examples of inputs/outputs.
    *   **Thorough Testing:** Validate components through unit, integration, and end-to-end tests.
    *   **Bias Review:** Regularly assess for potential biases in evaluation design.
    

## Inspect Framework-Specific Standards

These guidelines focus on leveraging Inspect’s features to implement evaluations that align with its requirements.

*   **Inspect-Specific Task Design**
    
    *   **Task Definition:**
        *   Use Inspect’s `Task` class to define evaluations, ensuring each task is modular and self-contained.
        *   Make tasks configurable. For any aspect of the task where changing it would likely result in an interesting evaluation, make that available as a top-level configuration option.
            *   For all configuration options, document the possible values
            *   If your code is structured so that your top-level object is a function `(...) → Task` , ensure that every argument to this function makes sense to configure the task, and everything reasonably configurable is an argument to this function.
    *   **Solver Composition:**
        *   Use built-in solvers where possible.
        *   Define custom solvers only when they materially enhance the evaluation.
    
*   **Dataset Integration**
    
    *   **Supported Formats:**
        *   Leverage Inspect’s native support for CSV, JSON, JSON Lines, and Hugging Face datasets.
        *   Use `FieldSpec` or custom readers for flexible field mapping.
    *   **Dynamic Resources:**
        *   Mock or cache external resources (e.g., APIs, datasets) to ensure reproducibility. For datasets, follow these caching guidelines:
            *   **Small Datasets (≤ 5MB):** Cache directly within the repository to simplify access and version control.
            *   **Large Datasets (> 5MB):** Store in a designated S3 bucket to manage storage efficiently and reduce repository bloat.
    *   **Testing:**
        *   Validate that datasets are correctly loaded and mapped to the `Sample` structure.
        *   Validate that datasets are reproducible (i.e. call `create_dataset()` twice and check that you get the same thing). This is likely only necessary if randomness is involved.
    
*   **Scoring Mechanisms**
    
    *   **Binary Metrics:**
        *   Use `ExactMatchScorer` or `ChoiceScorer` for primary binary metrics.
    *   **Auxiliary Metrics:**
        *   Add additional scorers to measure subtler capabilities (e.g., reasoning complexity).
    *   **Reproducibility:**
        *   Implement deterministic scorers to ensure consistent results.
    
*   **Observability and Logging**
    
    *   **Logging Standards:**
        *   Use `TaskState.store` for tracking key variables and avoid over-reliance on `metadata`.
        *   Record significant events with Inspect’s `@subtask` and `transcript` decorators.
    *   **QA Logs:**
        *   Include Inspect log files for QA in the designated `<evaluation_directory>/QA/` folder.
    
*   **Validation and QA**
    
    *   **Basic QA Requirements:**
        *   Provide evidence that the evaluation works with at least one sample.
        *   Test end-to-end functionality using frontier models or mocked APIs.
    *   **Custom Components:**
        *   Write unit tests for all custom solvers, tools, or external dependencies.
    *   **Frontier Model Testing:**
        *   Validate with a frontier model to identify edge cases or potential failure modes.
    
*   **Ethical and Safety Standards**
    
    *   **Safety Checks:**
        *   Avoid designing evaluations that encourage unsafe or unethical behavior.
        *   For risky capabilities, use simulations or subtasks to limit real-world implications.
    *   **Bias Mitigation:**
        *   Regularly review datasets, prompts, and scoring mechanisms for biases.
    
*   **Inspect Repository Standards**
    
    *   **Template Compliance:**
        *   Follow the repository structure and include required files like `README.md`, `CONTRIBUTING.md`, and `METADATA.json`.
        *   Pass all automated checks in the template repository.
    *   **Version Control:**
        *   Pin all dependencies and document all software versions for reproducibility.
            *   For Python dependencies used by the evaluation itself (i.e. not inside the sandbox), document the dependencies in `pyproject.toml` using [uv projects](https://docs.astral.sh/uv/guides/projects/).
    
*   **Testing and Debugging**
    
    *   **Unit Testing:**
        *   Ensure each custom tool, solver, and scorer has dedicated unit tests.
    *   **End-to-End Testing:**
        *   Use mocked LLM APIs for comprehensive evaluation testing without incurring high costs.
    *   **QA Logs Review:**
        *   Read logs to confirm the absence of errors and correct solver behavior.
    
*   **Documentation**
    
    *   **README.md**
        
        *   Ensure the README contains instructions for setting up the evaluation. It should be possible, by reading the README, to mechanically follow instructions from a freshly-cloned repo to running the evaluation.
            
    *   **CONTRIBUTING.md**
        
        *   Dependency maintenance should be documented here: where are dependency versions located? Where should they be updated?
            
        *   Tests should be documented here: where are tests located? How can I run the tests? What different kinds of tests are there?
            
        *   Any other maintenance tasks should be documented here, e.g. any external APIs, any cached datasets or tools that might need to be updated, etc.
            
    *   **Code**
        
        *   All code should be either self-documenting (i.e. the name of the variable, function or class is self-explanatory) or explicitly documented with docstrings and comments.
            
        *   All code should be type-annotated.
            
        *   Type annotations should be automatically verified by running `mypy` .
            

## **Best Practices Summary**

*   **Clarity and Consistency**
    
    *   Define clear objectives for each evaluation, with specific pass/fail criteria.
    *   Use consistent system prompts and instructions across tasks to ensure fair comparisons.
    *   Document all templates, datasets, and assumptions for transparency and reproducibility.
    
*   **Statistical Rigor**
    
    *   Naive vs clustered mean: Your dataset might be skewed towards certain question types/tasks. Cluster standard errors on the unit of randomization (for example, passage of text).
    *   Epochs: Use Inspect's built-in epoch parameter to resample answers from the same model several times and use the question-level averages as the question scores.
    *   On top of average scores, report the standard error of the mean.
    *   See more information [here](https://www.anthropic.com/research/statistical-approach-to-model-evals).
    
*   **Robust Dataset Management**
    
    *   Ensure datasets are clean, representative, and free from biases.
    *   Maintain strict separation between training and evaluation data.
    *   Use randomization and clustering to reduce order effects and handle grouped questions effectively.
    
*   **Efficient and Transparent Implementation**
    
    *   Log all evaluation steps for debugging and review.
    *   Optimize workflows by caching API calls and mocking external resources where feasible.
    
*   **Reproducible Scoring and Metrics**
    
    *   Use automated, reproducible scoring methods.
    *   Include both binary metrics (pass/fail) and richer auxiliary metrics to capture nuanced model performance.
    *   Validate scoring logic through unit tests and ensure edge cases are well-documented.
    
*   **Ethical and Safe Evaluation Design**
    
    *   Avoid encouraging unsafe or unethical behaviors. Use simulated environments for testing risky capabilities.
    *   Regularly review datasets, prompts, and scoring methods for biases or unintended consequences.
    
*   **Comprehensive Quality Assurance**
    
    *   Perform end-to-end testing of the full pipeline, using mocked inputs or a small subset of the dataset.
    *   Validate evaluations on frontier models and manually check results for anomalies.
    *   Provide QA logs demonstrating successful task completion for at least one sample per category.
