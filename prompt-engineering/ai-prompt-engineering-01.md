## Advanced Prompting

### 1. Build Self-Correction Systems
These techniques force the model to critique and refine its own output, pushing past the limitations of single-pass generation.

* **Chain of Verification**
    * **Description:** Requires a verification loop within the prompt, structuring the generation to include self-critique as a mandatory step.
    * **Example:** "Analyze this acquisition agreement, list three findings. **Now, identify three ways your analysis might be incomplete**, cite the specific language that confirms or refutes the concern, and then **revise your findings** based on this verification."

* **Adversarial Prompting**
    * **Description:** A more aggressive technique that demands the model actively find problems or attack its own previous design/output.
    * **Example:** "Please attack your previous design. You need to **identify five specific ways it could be compromised**. For each vulnerability, assess the likelihood and impact."

* **Strategic Edge-Case Learning (F-Shot Examples)**
    * **Description:** Include examples of common **failure modes** and boundary cases in your prompt to teach the model how to distinguish subtle gray spaces.
    * **Example:** To prevent SQL injection, provide a first example of an obvious raw string concatenation (baseline) and a second, more subtle example of a **parameterized query that looks safe** but has a second-order injection stored elsewhere.

***

### 2. Meta Prompting
This exploits the model's meta-knowledge by asking it to think about or optimize the prompt itself.

* **Reverse Prompting**
    * **Description:** Ask the model to design the single most effective prompt to solve a defined task, and then execute that self-designed prompt.
    * **Example:** "**You're an expert prompt designer.** Please design the single most effective prompt to analyze quarterly earnings reports for early warning signs of financial distress... **then execute that prompt** on this particular report."

* **Recursive Prompt Optimization**
    * **Description:** Define multiple iterative steps for the model to refine a prompt in a single pass, improving the quality on specific axes you care about.
    * **Example:** "My current prompt is here. **For version one, just add the missing constraints.** For version two, please resolve ambiguities. For version three, enhance reasoning depth."

***

### 3. Reasoning Scaffolds
This provides a structure that controls *how* the model thinks, forcing deeper and more comprehensive analysis.

* **Deliberate Over-Instruction**
    * **Description:** Explicitly fight the default compression of reasoning chains by demanding exhaustive depth and full reasoning.
    * **Example:** "Do not summarize. **Expand every single point with implementation details, edge cases, failure modes, and historical context.** Please prioritize completeness."

* **Zero-Shot Chain of Thought Structure**
    * **Description:** Provide a template with blank steps to trigger a chain of thought automatically, forcing problem decomposition.
    * **Example:** To root cause a technical issue, list the required steps as sequential questions with blanks: "1. What is the current system state? \[Answer\]. 2. What logs confirm the failure? \[Answer\]. 3. What is the most likely component failure? \[Answer\]..."

* **Reference Class Priming**
    * **Description:** Provide an example of high-quality reasoning (a quality benchmark) and ask the model to match that bar.
    * **Example:** Provide the model with one of its **own best outputs of a detailed analysis** and then say, "Please provide your analysis for this new data, ensuring the depth and quality of reasoning matches the standard set by the example provided below."

***

### 4. Perspective Engineering
This counters single-perspective blind spots by simulating competing viewpoints and different reasoning modes.

* **Multi-Persona Debate**
    * **Description:** Instantiate experts with specific, potentially **conflicting priorities** to simulate a rigorous debate before synthesizing a recommendation.
    * **Example:** "**Expert 1 has Priority X (Cost Reduction). Expert 2 has Priority Y (Security).** They must argue for their preference and critique the others' positions. After debate, you must synthesize a recommendation that addresses all concerns."

* **Temperature Simulation**
    * **Description:** Have the model roleplay at different "temperatures" (deterministic vs. creative) for different passes on the same problem.
    * **Example:** "I want a **junior analyst who is uncertain and overexplains** to look at this problem first. Then, I want a **confident expert who is concise and direct**. Finally, synthesize both perspectives and highlight where uncertainty is warranted."
