
Great tweet: https://x.com/_jasonwei/status/1712551642275655770

# Introduction: The Evolving Chain-of-thought

There's no shortage of prompting techniques. Some are more widely adopted than others. In this post I focus on papers from the last year or so. Most of these paper focus on techniques that aim to improve upon the commonly used chain-of-thought (CoT) approach, especially for tasks requiring complex reasoning. It's not an exhaustive list or deep dive into each paper, but rather my summaries from paper I've been looking into recently. 

As models become more sophisticated, one might wonder: Is prompt engineering still relevant? While newer models may have internalized many good prompting practices, there's still a vast ecosystem of models and tasks that benefit greatly from well-crafted prompts. 

It's important to note that many of these papers used weaker LLMs than today's top closed models. As such, the specific results reported may not be directly reproducible so I put less emphasis on the results reported. 

Before we dive in, let's recall a basic example of a Chain-of-Thought (CoT) exampler (or demonstartion):
Chain-of-though paper: Chain-of-Thought Prompting Elicits Reasoning in Large Language Models
https://proceedings.neurips.cc/paper_files/paper/2022/file/9d5609613524ecf4f15af0f7b31abca4-Paper-Conference.pdf

```
Q: Take the last letters of the words in "Willie Valeria Zane" and concatenate them.
A: Let's think step by step.

The last letter of "Willie" is "e".
The last letter of "Valeria" is "a".
The last letter of "Zane" is "e".
Concatenating them gives us: "eae".
Therefore, the answer is eae.
```

# Let's Try Again Approaches 

Sonnet 3.5 Intro:
"Ever felt like you needed a do-over on a task? Well, these approaches give LLMs that chance. They're all about generating multiple responses at inference time, not to be confused with fancy decoding strategies. Let's check out two cool techniques: Self-Consistency and Self-Refine."

### Self Refine (Neurips 2023)

Iterative Refinement with Self-Feedback https://proceedings.neurips.cc/paper_files/paper/2023/file/91edff07232fb1b55a505a9e9f6c0ff3-Paper-Conference.pdf
Code: https://github.com/madaan/self-refine

Self-Refine is all about the LLM giving itself feedback. Here's the gist:

1. Generate an initial response (initial prompt)
2. The LLM then critiques its own output (using a specific feedback prompt)
3. Use that feedback to refine the original prompt 
4. Generate a new, hopefully improved response
5. Rinse and repeat

They key to this approach is a Feedback prompt that is guided only by instructions and few-shot examples. Using a single LLM throughout. The refined prompt incorporates the history of previous iterations. Essentially, this method automates the iterative prompt refinement typically performed manually by users.

This is one of the only paper that actually conducted experiments with GPT-4, showing performance gains across datasets, with Math Reasoning showing the least gains. They claim it's due to GPT-4's tendency to overestimate the correctness of its mathematical outputs. They also point out that the most substantial improvements occur in the early iterations. This is important since runing many iterations is slow and expensive. Another important point is that the feedback prompt is task-specific, tailored to each individual task being performed. This tailoring, while beneficial, may lead to biased results if the prompt becomes overly attuned to the particular errors the LLM tends to make on the test set used.

Let's see an example. One of the tasks in which they observed the highest gains compared to the base models is Constrained Generation, where the model is asked to generate a coherent sentence containing some input concepts given by the user. This example is based on few-shot prompting to guide model outputs, but they claim that Self Refine works in zero-shot setups as well. 

Initial prompt:

```
# Example 1
Concepts: ['footage', 'motion', 'ruin', 'tilt', 'window']
Sentence: time lapse footage with tilt up motion of the sun streaking through window of ruin
# Example 2
Concepts: ['cause', 'hate', 'hut', 'local', 'love']
Sentence: new beach huts on the island have caused some controversy some locals love them others hate them

Concepts: ['card', 'chip', 'deal', 'dealer', 'table']
```

The feeback prompt targets generations with missing concepts:

```
Concepts: ['beat', 'drum', 'pen', 'sit', 'use']
Sentence: A man uses a drum to beat a pen.
what concepts from the concept list are missing from the sentence?
Feedback: sit

Concepts: ['chair', 'clipper', 'cut', 'hair', 'sit']
Sentence: A girl sitting on the couch with her hair up.
what concepts from the concept list are missing from the sentence?
Feedback: clipper, cut, chair
```

Here's the refined prompt after two iterations of generation + feedback:

```
# Examples ...

Concepts: ['card', 'chip', 'deal', 'dealer', 'table']
Sentence: a dealer offers a card to a group of people at a table  # First response

# Feedback step:

Concepts: ['card', 'chip', 'deal', 'dealer', 'table']
Sentence: a dealer offers a card to a group of people at a table
what concepts from the concept list are missing from the sentence?
Feedback: chip, deal  # The first feedback response

Okay, impove the sentence using the sentence:

Sentence: The dealer dealt each person at the table and offered them a card.  
what concepts from the concept list are missing from the sentence?
Feedback: chip  # The second feedback response

Okay, impove the sentence using the sentence:

Sentence: The dealer dealt each person at the table a chip and offered them a card.
what concepts from the concept list are missing from the sentence?
Feedback: NONE  # # The third feedback response
```

In this example, the first response was `a dealer offers a card to a group of people at a table` and after a couple of iterations we ended up with an improved version `The dealer dealt each person at the table a chip and offered them a card.`.

### Is the Feedback/Refinment step important?

Can we just generate multiple outputs instead of refining using feedback? The authors ran an experiment to validate this. They used the LLM to generate 4 samples (without any feedback and refinment) and asked humans to annotate if a single reponse generated by Self-Refine is better than *all* 4 samples genreated. They reported scores only for two tasks - acronym generation and sentiment reversal. Their results clearly show that humans preferred the Self-Refine generations more often. However, they don't mention how they generated the multiple samples. It seems they used ChatGPT so I'm assuming they just ran it 4 times per input, rather then trying a more common sampling approach. Also, based on Figure 7 in the paper, it seems that ChatGPT with a single reponse performed better than this multi-sample approach on this experiment. Experimental setup matters. 

This experiment is actually a good segway to the next paper. 


### Self Consistency (ICLR 2023)
SELF-CONSISTENCY IMPROVES CHAIN OF THOUGHT
REASONING IN LANGUAGE MODELS
https://arxiv.org/pdf/2203.11171
Code: 

The authors nicely refer to this appraoch as "it acts more like a “self-ensemble” that works on top of a single language model".  

1. Start with a prompt containing some manually written chain-of-thought examples.
2. Instead of generating just one response, we ask for several using different sampling techniques (like temperature sampling).
3. Finally, we use majority voting to pick the most "consistent" answer. (there's a task specific parser to extrat the answer from each response such as parsing the first numerical part after the model generates “The answer is ")

There are different sampling techniques to try. They did an ablation study and concluded that self-consistency is generally robust to sampling strategies and parameters.

Importantly, they also showed that self consistency works better than beam search decoding with the same number of beams. Their explanation is that self consistency with sampling generates more diverse samples than beam search. 

Limitation - while they showed gains with 5-10 samples, the best results come from using around 40 samples. That's a lot of generations, which can get pretty expensive and slow. Not very practical.

 I've seen many papers apply self consistency with their approach to improve results further. It seems widely known but maybe not widely adopted in practice. 

Some relevant related work: Reflexion (Shinn et al., 2023) and Self-Correction (Welleck et al., 2022). 


## Active Prompting

## Paper: Active Prompting with Chain-of-Thought for Large Language Models (ACL 2024)
### Github: https://github.com/shizhediao/active-prompt

Sonnet 3.5 Intro:
"Ever tried to come up with perfect examples for few-shot Chain-of-Thought prompts? It's not as easy as it sounds, right? You want examples that'll really boost the model's performance, but finding them can be a head-scratcher. Well, some clever folks have been working on automating this process, and it's pretty cool stuff."

Writing CoT examples isn't always easy. You want good examples that will improve the model performance. Like examples that the LLM doesn't handle well yet. This paper automates this step, inspired by active learning from more traditional ML. Here's how it works:
1. Run each example or task through the LLM multiple times.
2. Look for examples where the LLM gives inconsistent answers. (uncertanity estimation step)
3. Annotate these examples and add them to your prompt.

This is nice because you don't need to have labels in order to find examples where the LLM struggles with, and all the extra computation is done offline to build the prompt. 

How is uncertanity esimtated? They experiment with different approaches and Disagreemnt and Entropy consistently showed the best performance. Disagreemnt is simple, it's based on the number of unique answers generated by the model across multiple runs, divided by the total number of runs. Entropy is similar but it also considers that distribution of the unique answers. Two unique answers distributed equally in 10 samples would have a higher entropy (higher uncertanity) than two unique answers with 9/10 samples the same (Disagreement would be the same for both cases).    

They also tried using the model's self confidence (asking the model to evaluate its own confidence in its answer). They found it performed poorly as the model tends to be too confident in its generation. A better approach was to use the model's log probaiblities but that is not always available. 

Extra read: Auto-CoT that also aims to improve examplers diversity automatically. It samples questions with diversity and generates reasoning chains to construct CoT examplers. It takes it a step further as it alo annotate the exampler unlike in the adaptive prompt work.  
"Automatic chain of thought prompting in large language models"

## From natural language to code

There has been a lot of work on code generation from natural lanauge using LLMs, like SQL, Python, etc. Two papers with that came out about the same time follow this paradigm to help with various tasks.  

### Program of Thoughts
Program of Thoughts Prompting: Disentangling Computation from Reasoning for Numerical Reasoning Tasks
https://reasonwithpal.com
Github: https://github.com/TIGER-AI-Lab/Program-of-Thoughts

For tasks that require numerical computation, the idea is to let the LLM do the reasoning and language understanding, and delegate the actual computation to a generated program. The motivation for this is that LLMs are prone to arithmetic calculation erros, and struggle wtih complex computations, but are good at generating code. An LLM with a calculator as a tool is an example of a way to solve tasks that require simple arthmetics. 
Works in this area mostly differ is in their intermidate representation, which could be a set of equations, Python code, pseudo code, etc. 



<!-- 
They stress out that PoT is not similar to equations generation:
"
The ‘program of thoughts’ is different from generating equations directly, where the generation target would
be solve(20000 ∗ (1 +x)
3 −2000−x∗ 20000 ∗ 3−1000, x). As observed by Wei et al. (2022) for CoT, directly
generating such equations is challenging for LLMs. PoT differs from equation generation in two aspects: (1)
PoT breaks down the equation into a multi-step ‘thought’ process, and (2) PoT binds semantic meanings
to variables to help ground the model in language. We found that this sort of ‘thoughtful’ process can
elicit language models’ reasoning capabilities and generate more accurate programs. We provide a detailed
comparison in the experimental section.
"

PoT as an Intermediate Step: Important. When the task requires additional textual reasoning, PoT can be combined with CoT. 

"During demonstration, we present LLMs with examples to teach it predict whether to an additional CoT
reasoning needs to be used. If LLM outputs ‘keep prompting’ in the end, we will adopt the execution results
from PoT as input to further prompt LLMs to derive the answer through CoT." 
"Please note that this prompting strategy is only needed for the AQuA because the other datasets can all be solved by PoT-only prompting."


To execute programs that use sympy https://www.sympy.org/en/index.html. 


"
We also leverage an external calculator as suggested in Wei et al. (2022) for all the equations
generated by CoT, which is denoted as CoT + calc. Besides greedy decoding, we use self-consistency (Wang
et al., 2022b) with CoT, taking the majority vote over 40 different completions as the prediction.
"

Question from me: Can we just implement a few simple arthmitic functions for our needs such as this paper did: https://proceedings.neurips.cc/paper_files/paper/2022/file/9d5609613524ecf4f15af0f7b31abca4-Paper-Conference.pdf

Important to mention: Why not just give CoT a calculator?

"As an ablation, we also compare with CoT+calc, which leverages an external calculator to correct
the calculation results in the generated ‘chain of thoughts’. The experiments show that adding an external
calculator only shows mild improvement over CoT on MWP datasets, much behind PoT. The main reason
for poor performance of ‘calculator’ is due to its rigid post-processing step, which can lead to low recall in
terms of calibrating the calculation results."

They claim to be better than PaL: 
"Comparison with PaL We also compare PoT with another more recent related approach like PaL (Gao
et al., 2022). According to to Table 5, we found that our method is in general better than PaL, especially
on SVAMP and ASDIV. Our results are 6% higher than their prompting method."


This means that for simple arthimetic CoT with calc is probably good enough. 

"Another concurrent work similar to ours is BINDER (Cheng et al., 2022), which applies Codex to synthesize
‘soft’ SQL queries to answer questions from tables."

"Recently, there has been several follow-up work on top of PoT including self-critic (Gou et al., 2023), selfeval (Xie et al., 2023), plan-and-solve (Wang et al., 2023a). These methods propose to enhance LLMs’
capabilities to solve math problems with PoT. self-critic (Gou et al., 2023) and self-eval (Xie et al., 2021)
both adopt self-evaluation to enhance the robustness of the generated program. plan-and-solve (Wang et al.,
2023a) instead adopt more detailed planning instruction to help LLMs create a high-level reasoning plan.
These methods all prove to bring decent improvements over PoT on different math reasoning datasets."

Similar to tool calls:

"Another line of work related to ours is Tool-use in transformer models (Schick et al., 2023; Paranjape et al.,
2023). These work propose to adopt different tools to help the language models ground on external world.
These work generalizes our Python program into more general API calls to include search engine, string
extraction, etc. By generalization, LLMs can unlock its capabilities to solve more complex reasoning and
grounding problems in real-world scenarios."

Limitations: You don't want the LLM to execute some crazy code like  ‘import os; os.rmdir()’... Need to block it from doing something stupid.
"Another limitation is
that PoT still struggles with AQuA dataset with complex algebraic questions with only 58% accuracy. It’s
mainly due to the diversity questions in AQuA, which the demonstration cannot possibly cover." -->

This work doesn't use a specific programming language. It breaks the problem into a multi-step ‘thought’ process and binds semantic meanings to variables to help ground the model in language. Let's see how it works for a typical math word problem. 

```
Josh decides to try flipping a house. He buys a house for $80,000 and then puts in $50,000 in repairs.
This increased the value of the house by 150%. How much profit did he make?
```

Note that GPT-4o (as of Sep 20, 2024) struggles with this question and answers it incorrectly. It fails to generate the correct equations. In a typical chain-of-thought reasining the demonstartion would like something like this:

```
Let's break it down step by step. Josh bought the house for $80,000. He spent $50,000 on repairs. This means that his Total investment is $80,000 + $50,000 = $130,000...
```

As you can see, the LLM does everything, including the calculations. In program of thoughts:

```
Question: Josh decides to try flipping a house. He buys a house for $80,000 and then puts in $50,000 in repairs.
This increased the value of the house by 150%. How much profit did he make?
# Python code, return ans
cost_of_original_house = 80000
increase_rate = 150 / 100
value_of_house = (1 + increase_rate) * cost_of_original_house
cost_of_repair = 50000
ans = value_of_house - cost_of_repair - cost_of_original_house
```

Then the solve the program using the SymPy library https://www.sympy.org/en/index.html. 
This is the general idea. You can desing more complex demonstartions where the program as an intermidate step. 


### PAL: Program-aided Language Models.

Very similar to PoT but generates valid python code. The LLM generates a python program as its reasining step and offloads the solution to a python interpreter. They also focus on CoT-style reasoning chains. LLMs knwo Python well so python is a more preferable representation. 

Nice experiment they did is putting the answers in the prompt to test if the LLM can actually replace the interperter. They found that results dropped signficantly. Which shows the improvement is not just from the new prompt style. Also, they found that meaningful variable names are important than using random names, as you might expect. 


Prompt example.

```
# Q: I have a chair, two potatoes, a cauliflower, a lettuce head, two tables, a
cabbage, two onions, and three fridges. How many vegetables do I have?
# note: I'm not counting the chair, tables, or fridges
vegetables_to_count = {
'potato': 2,
'cauliflower': 1,
'lettuce head': 1,
'cabbage': 1,
'onion': 2
}
print(sum(vegetables_to_count.values()))
```


Related work: BINDING LANGUAGE MODELS IN SYMBOLIC LANGUAGES
https://github.com/xlang-ai/Binder
ICLR 2023
https://arxiv.org/pdf/2210.02875


## 

# In-Context Principle Learning from Mistakes (LEAP)

This work actually used gpt-4 which is nice. 

This is another nice approach that helps automate the process of writing useful instructions in the prompt (“principles”). We all wrote or saw prompts with many lines of instrctions, DOs and DON'Ts, etc. The high-level LEAP (Learning Principles) approach goes like this:

- Run the LLM in zero-shot CoT fashion multiple times on some questions you have gold answers for
- Ask the LLM to evaluate answers against the gold 
- For each mistake, ask the LLM to generate low-level principles
- Generate high-level principles from the low-level principles
- Create an enhanced prompt with the low-level and/or high-level principles, with optional few-shot examples (recommended). This is the prompt to be used to label new unseen examples. 

Low-level prompt:
```
Question: {question}
Generated Reasoning: {response}
Generated Answer: {generated_answer}
Correct Reasoning: {correct_reasoning}
Correct Answer: {correct_answer}
Instruction: Conduct a thorough analysis of the generated answer in comparison to the
correct answer. Also observe how the generated reasoning differs from the correct
reasoning. Identify any discrepancies, misunderstandings, or errors. Provide clear
insights, principles, or guidelines that can be derived from this analysis to improve
future responses. We are not focused on this one data point, but rather on the general
principle.
Reasoning: <discuss why the generated answer is wrong>
Insights: <what principle should be looked at carefully to improve the performance in
the future>
```

Low-level examples:

```
1. The system should be designed to understand the context of the question better. In this case, it should
have recognized that the question was asking for the duration of existence of the European Coal and Steel
Community before it transitioned into the European Economic Community. Understanding the specific
context and requirements of a question is crucial for generating accurate answers
```



High-level princpiles prompt:
````
Low-level principles: {low_level_principles}
Create a list of *unique* and insightful principles to improve future responses based
on the analysis above.
Focus on capturing the essence of the feedback while eliminating redundancies.
Ensure that each point is clear, concise, and directly derived from the introspection
results.
Create a numbered list of principles. Leave specific details in place.
Limit to at most 8 principles.
List of Principles:
````

Hihg-level princples examples:

1. Ensure clarity and precision: Responses should be clear and concise, avoiding any ambiguity or unnecessary complexity.
2. Maintain relevance: Responses should directly address the query or topic at hand, avoiding any unrelated
or tangential information.
1. Prioritize uniqueness: Strive to provide unique insights or perspectives in responses, avoiding repetition or
common knowledge.

Final prompt:

```
Instruction: {instruction}
In doing so, please carefully note the following principles:
Principles: {principles}
---
{few_shot_questions_and_answers}
Q: {test_question}
````

The authors show improvements on some datasets over CoT few-shot but not on all. With  Llama-2-chat-70B for example, they couldn't see a benefit from the generated principles, even though they used GPT-4 to generate the principles. Also, for the strong models they evaluated (GPT-4, GPT-4 Turbo) the gains were pretty modest for the most part if at all. 

Which principles provide more value?  "We could not identify any particular pattern as to which
method should be used. We thus
suggest that in real-life scenarios, both approaches should
be tested, and selected using a validation set"

## Plan and Solve One by One Approaches

# least-to-most prompting (Zhou et al., 2022) 


# dynamic least-to-most prompting
LEAST- TO -M OST P ROMPTING E NABLES C OMPLEX R EASONING IN L ARGE L ANGUAGE M ODELS
"Chain-of-thought prompting has demonstrated remarkable performance on various natural language reasoning tasks. However, it tends to perform poorly on tasks which requires solving problems harder than the exemplars shown in the prompts. To overcome this challenge of easy-to-hard generalization, we propose a novel prompting strategy, least-to-most prompting."

Simple idea. All steps happen in sequence:
1. Decompose the task into subproblems using few-shot prompting
2. Solve every sub-problem using few-shot prompting. Every prompt include the response of it prior sub-problems

"Least-to-most prompting teaches language models how to solve a complex problem by decomposing it to a series of simpler subproblems. It consists of two sequential stages:

1. Decomposition. The prompt in this stage contains constant examples that demonstrate the decomposition, followed by the speciﬁc question to be decomposed.

2. Subproblem solving. The prompt in this stage consists of three parts: (1) constant examples demonstrating how subproblems are solved; (2) a potentially empty list of previously answered subquestions and generated solutions, and (3) the question to be answered next."
    
The main limitation is inefficency. You need to make multiple LLM calls sequntially. The other limitation is that you need a lot of prompting and trail and error to get the prompts working well for your problem and domain. 

One of the tasks they got 100% accuracy on is last-letter concatenation with text-davinci-002. Decomposing is straightforward for this task which helps the it do well on it. They tested up to size 12 (which is the maximum that we tested for our experiment) even though it only contains one exemplar each for lists of sizes 2 and 3.

Standard prompt:

Q: “think, machine” A: “ke”

Chain-of-thought:

Q: “think, machine” A: The last letter of “think” is “k”. The last letter of “machine” is “e”. Concatenating “k”, “e” leads to “ke”. So, “think, machine” outputs “ke”.


L EAST- TO - MOST PROMPTING

Q: “think, machine”
A: The last letter of “think” is “k”. The last letter of “machine” is “e”. Concatenating “k”, “e” leads to “keg”. So, “think, machine, learning” outputs “keg”.


“worm, jackson, widow, car” Golden:

“mnwr” Solution:

Q: “worm, jackson” A: The last letter of “worm” is “m”. The last letter of “jackson” is “n”. Concatenating: “m”, “n” leads to “mn”. So, “worm, jackson” outputs “mn”.

Q: “worm, jackson, widow” A: “worm, jackson” outputs “mn”. The last letter of “widow” is “w”. Concatenating: “mn”, “w” leads to “mnw”. So, “worm, jackson, widow” outputs “mnw”.

Q: “worm, jackson, widow, car” A: “worm, jackson, widow” outputs “mnw”. The last letter of “car” is “r”. Concatenating: “mnw”, “r” leads to “mnwr”. So, “worm, jackson, widow, car” outputs “mnwr”.


Q: Among the other denominations there are approximately 93,000 Mormons in 253 congregations, 25,000 Jewish adherents in 21 synagogues, 12,000 Muslims in 39 masjids, 7,000 Buddhists in 34 temples, 7,000 Hindus in 17 temples, 2,500 Unitarian Universalism in 9 congregations, 2,000 Bahai in 17 temples, 5 Sikh temples, a Zoroastrian temple, a Jain temple and an uncounted number of neopagans. How many more Mormons were there compared to Muslims?

Chain-of-thought by code-davinci-002 A: According to the passage, there are 93,000 Mormons and 25,000 Muslims. 93,000 - 25,000 = 68,000. So the answer is 68,000.

Least-to-most by code-davinci-002 Q: How many Mormons were there? A: “93,000 Mormons”. So the answer is 93,000.

Q: How many Muslims were there?

A: “12,000 Muslims”. So the answer is 12,000.

Q: How many more Mormons were there compared to Muslims?

A: We know that there were 93,000 Mormons. We also know that there were 12,000 Muslims. So there were 93,000 - 12,000 = 81,000 more Mormons than Muslims. So the answer is 81,000.

Analysis wrong.

another prompt example:

10.3.1 C HAIN - OF -T HOUGHT (1- SHOT )

Q: Elsa has 5 apples. Anna has 2 more apples than Elsa. How many apples do they have together? A: Anna has 2 more apples than Elsa, so Anna has 2 + 5 = 7 apples. Elsa and Anna have 5 + 7 = 12 apples together. The answer is 12.

10.3.2 L EAST- TO -M OST (1- SHOT )

Q: Elsa has 5 apples. Anna has 2 more apples than Elsa. How many apples do they have together? A: Let’s break down this problem: 1. How many apples does Anna have? 2. How many apples do Elsa and Anna have together?

1. Anna has 2 more apples than Elsa. So Anna has 2 + 5 = 7 apples.

2. Elsa and Anna have 5 + 7 = 12 apples together.

Q: { question } A: Let’s break down this problem: —The answer is:

# Plan-and-Solve Prompting

This is pretty much the same as least-to-most prompting just with a different name. 

". In our experiments, we simply replace “Let’s think step by step” of Zeroshot-CoT with “Let’s first understand the problem and devise a plan to solve the problem. Then, let’s carry out the plan and solve the problem step by step”

To address the calculation errors of Zero-shotCoT and improve the quality of generated reasoning steps, we add more detailed instructions to PS prompting. Specifically, we extend it with “extract relevant variables and their corresponding numerals” and “calculate intermediate results (pay attention to calculation and commonsense)” instructions. This prompting variant is called the PS+ prompting strategy (see Figure 3 (b)). Despite its simplicity, PS+ strategy greatly improves the quality of the generated reasoning process."

They seem to focus on fewshot PS+. "Similar to Zero-shot-CoT, Zero-shot PS prompting consists of two steps. In step 1, the prompt first makes an inference using the proposed prompting template to generate the reasoning process and the answer to a problem. In step 2, it extracts the answer for evaluation by using the answer extraction prompting, such as “Therefore, the answer (arabic numerals) is”." This means it's more efficent than least-to-most. 

"• The templates should elicit LLMs to determine subtasks and accomplish the subtasks." -- but they do it all at once, unlike least-to-most. 

"To meet the first criterion, we follow Zero-shotCoT and first convert the input data example into a prompt with a simple template “Q: [X]. A: [T]” Specifically, the input slot [X] contains the input problem statement and a hand-crafted instruction is specified in the input slot [T] to trigger LLMs to generate a reasoning process that includes a plan and steps to complete the plan." Thus, the prompt would be “Q: [X]. A: Let’s first understand the problem and devise a plan to solve the problem. Then, let’s carry out the plan and solve the problem step by step.” (PS, not PS+) vs "‘Let’s think step by step”. 

Step 2 prompt to extract the answer: "This prompt includes the answer extraction instruction appended to the first prompt followed by the LLM generated reasoning text. This way, LLM is expected to return the final answer in the desired form."

greedy decoding strategy = (1 output chain)

They mention Auto-CoT: "To leverage the benefit of demonstration examples and minimize manual effort, Zhang et al. (2022) designed Auto-CoT. It first automatically obtains k examples by clustering the given dataset. It then follows Zero-shot-CoT to generate rationales for the selected examples. Finally, demonstration examples are constructed by adding the generated rationales to selected examples as CoT prompts."

They also got signficant improvement with self-consistency. 
"Self-consistency (Wang et al., 2022b) (SC) is proposed to reduce randomness in LLM’s output by generating N reasoning results and determining the final answer by majority voting. Figure 4 shows that PS+ prompting with SC (73.7% and 84.4%) substantially outperforms that without SC (58.7% and"75.7%) on GSM8K and SVAMP, respectively.

In they related work they describe their main contribution over other works:
"Our work is different from the above works by focusing on eliciting multi-step reasoning by LLMs in a zero-shot approach. We ask LLMs to write a plan to decompose a complex reasoning task into multiple reasoning steps. Furthermore, we introduce detailed instructions to the prompt to avoid obvious errors in the reasoning steps. We refer readers to the survey (Huang and Chang, 2022) for more related works."

One issue of such works is that the authors showed the slight changes to how to modify the prompt can have significant impact on the results. This might be a problem since they used weaker models. 


# Results

Comparison of all approaches on a few datasets. 



Datasets:

- Arithmetic Reasoning: 
  (1) the GSM8K (Cobbe et al., 2021) dataset of high quality linguistically diverse grade school math word problems created by human problem writers,
  (2) AQUA (Ling et al., 2017) dataset of algebraic word problems with natural language rationale

- Commonsense Reasoning:
  (1) StrategyQA (Geva et al., 2021) benchmark dataset with questions requiring multi-step reasoning but the reasoning steps are not given. Hence, they are to be inferred;

- Symbolic Reasoning: (9) the Last Letter Concatenation (Wei et al., 2022b) dataset of questions requiring the last letters of words in a name to be concatenated (e.g., “James Brown” → “sn”)


# Final Thoughts on Prompt Engineering




ANother example where sonnet 3.5 did well on but gpt failed:

Update the following code to take the input file from the command line instead. Note the code hard codes the name in more than one place. 

```
import ast
import json
import os


def convert_single_quotes_to_double_quotes(file_path):
    with open(file_path, 'r') as f:
        content = f.read()
        try:
            # Safely evaluate the single-quoted string using ast.literal_eval
            data = ast.literal_eval(content)
            return data
        except Exception as e:
            print(f"Error converting {file_path}: {e}")
            return None


def convert_json_files_to_jsonl(folder_path, output_file):
    with open(output_file, 'w') as jsonl_file:
        for filename in os.listdir(folder_path):
            if filename.endswith('HOV.json'):
                file_path = os.path.join(folder_path, filename)
                
                # Convert the single quotes to double quotes and retrieve valid JSON
                data = convert_single_quotes_to_double_quotes(file_path)
                
                # If data is valid, write it to the JSONL file
                if data:
                    if isinstance(data, list):
                        for entry in data:
                            jsonl_file.write(json.dumps(entry) + '\n')
                    elif isinstance(data, dict):
                        jsonl_file.write(json.dumps(data) + '\n')


# Get the folder path where this script is located
folder_path = os.path.dirname(os.path.realpath(__file__))
output_file = os.path.join(folder_path, 'HOV-queries.jsonl')
convert_json_files_to_jsonl(folder_path, output_file)

```
# Framework

Dspy. 