+++
title = 'Beyond Chain-of-Thought: Evolving Prompts for Enhanced LLM Reasoning'
date = 2024-09-24
draft = true
+++

# Introduction: The Evolving Chain-of-thought

There's no shortage of prompting techniques. Some are more widely adopted than others. In this post I focus on papers from the last year or two. Most of these papers focus on techniques that aim to improve upon the commonly used [chain-of-thought](https://proceedings.neurips.cc/paper_files/paper/2022/file/9d5609613524ecf4f15af0f7b31abca4-Paper-Conference.pdf) (CoT) approach, especially for tasks requiring complex reasoning. It's not an exhaustive list or deep dive into each paper, but rather my summaries of papers I've been looking into recently. 

As models become more sophisticated (o1-preview, I'm talking to you), one might wonder: How relevant is prompt engineering today? While newer models may have internalized many good prompting practices, there's still a vast ecosystem of models and tasks that benefit greatly from well-crafted prompts. AI is evolving fast and so is prompt engineering. 

It's important to note that many of these papers used weaker LLMs than today's top closed models. As such, I put less emphasis on the results reported. 

Before we dive in, let's recall a basic example of a CoT demonstartion:

```
Q: Take the last letters of the words in "Willie Valeria Zane" and concatenate them.
A: Let's think step by step. The last letter of "Willie" is "e". The last letter of "Valeria" is "a". The last letter of "Zane" is "e". Therefore, the answer is eae.
```

It only needs one pass through the language model. This efficiency, combined with its impressive performance on a variety of tasks, has made this simple technique the go-to choice for prompting. Can we do better? let's dive in. 

# Let's Try Again Approaches 

Sonnet 3.5 Intro:
"Ever felt like you needed a do-over on a task? Well, these approaches give LLMs that chance. They're all about generating multiple responses at inference time, not to be confused with fancy decoding strategies. Let's check out two cool techniques: Self-Consistency and Self-Refine."

## Self Refine (Neurips 2023)

[Iterative Refinement with Self-Feedback](https://proceedings.neurips.cc/paper_files/paper/2023/file/91edff07232fb1b55a505a9e9f6c0ff3-Paper-Conference.pdf)

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

## Self Consistency (ICLR 2023)
Paper: [Self-Consistency Improves Chain of Thought Reasoning in Language Models](https://arxiv.org/pdf/2203.11171)

Code: N/A

The authors nicely refer to this appraoch as "it acts more like a “self-ensemble” that works on top of a single language model".  

1. Start with a prompt containing some manually written chain-of-thought examples.
2. Instead of generating just one response, we ask for several using different sampling techniques (like temperature sampling).
3. Finally, we use majority voting to pick the most "consistent" answer. (there's a task specific parser to extrat the answer from each response such as parsing the first numerical part after the model generates “The answer is ")

There are different sampling techniques to try. They did an ablation study and concluded that self-consistency is generally robust to sampling strategies and parameters.

Importantly, they also showed that self consistency works better than beam search decoding with the same number of beams. Their explanation is that self consistency with sampling generates more diverse samples than beam search. 

Limitation - while they showed gains with 5-10 samples, the best results come from using around 40 samples. That's a lot of generations, which can get pretty expensive and slow. Not very practical.

 I've seen many papers apply self consistency with their approach to improve results further. It seems widely known but maybe not widely adopted in practice. 

Some relevant related work: Reflexion (Shinn et al., 2023) and Self-Correction (Welleck et al., 2022). 


# Active Prompting (ACL 2024)

Paper: [Active Prompting with Chain-of-Thought for Large Language Models](https://arxiv.org/pdf/2302.12246)

Code: https://github.com/shizhediao/active-prompt

Sonnet 3.5 Intro:
"Ever tried to come up with perfect examples for few-shot Chain-of-Thought prompts? It's not as easy as it sounds, right? You want examples that'll really boost the model's performance, but finding them can be a head-scratcher. Well, some clever folks have been working on automating this process, and it's pretty cool stuff."

Writing CoT examples isn't always easy. You want good examples that will improve the model performance. Like examples that the LLM doesn't handle well yet. This paper automates this step, inspired by active learning from more traditional ML. Here's how it works:
1. Run each example or task through the LLM multiple times.
2. Look for examples where the LLM gives inconsistent answers. (uncertanity estimation step)
3. Annotate these examples and add them to your prompt.

This is nice because you don't need to have labels in order to find examples where the LLM struggles with, and all the extra computation is done offline to build the prompt. 

How is uncertanity esimtated? They experiment with different approaches and Disagreemnt and Entropy consistently showed the best performance. Disagreemnt is simple, it's based on the number of unique answers generated by the model across multiple runs, divided by the total number of runs. Entropy is similar but it also considers that distribution of the unique answers. Two unique answers distributed equally in 10 samples would have a higher entropy (higher uncertanity) than two unique answers with 9/10 identical samples (Disagreement would be the same in both cases).    

They also tried using the model's self confidence (asking the model to evaluate its own confidence in its answer). They found it performed poorly as the model tends to be too confident in its generation. A better approach was to use the model's log probaiblities but that is not always available. 

Extra read: Auto-CoT that also aims to improve examplers diversity automatically. It samples questions with diversity and generates reasoning chains to construct CoT examplers. It takes it a step further as it alo annotate the exampler unlike in the adaptive prompt work.  
"Automatic chain of thought prompting in large language models"


# From Natural Language to Code
Sonnet 3.5 Intro: "Ever wished your AI assistant could code like a pro? Well, get ready! We're exploring some cool research that's bridging natural language and code. These clever approaches let LLMs handle the thinking while actual programs do the number-crunching. It's like giving your AI a super-smart calculator – let's dive in!"

There has been a lot of work on code generation from natural lanauge using LLMs, like SQL, Python, etc. Two papers with that came out about the same time follow this paradigm to help with various tasks.  

## Program of Thoughts (TMLR 2023)
Paper: [Program of Thoughts Prompting: Disentangling Computation from Reasoning for Numerical Reasoning Tasks](https://arxiv.org/pdf/2211.12588)

Github: https://github.com/TIGER-AI-Lab/Program-of-Thoughts

For tasks that require numerical computation, the idea is to let the LLM do the reasoning and language understanding, and delegate the actual computation to a generated program. The motivation for this is that LLMs are prone to arithmetic calculation erros, and struggle wtih complex computations, but are good at generating code. An LLM with a calculator as a tool is an example of a way to solve tasks that require simple arthmetics. 
Works in this area mostly differ is in their intermidate representation, which could be a set of equations, Python code, pseudo code, etc. 

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


## Program-aided Language Models (Poster, ICML 2023)
Paper: [PAL: Program-aided Language Models](https://arxiv.org/abs/2211.10435)
Code: https://reasonwithpal.com

Very similar to PoT but generates valid python code. The LLM generates a python program as its reasining step and offloads the solution to a python interpreter. They also focus on CoT-style reasoning chains. Python is a preferable representation for LLMs for obvious reasons. 

One nice experiment they performed was including the answers in the prompt to assess whether the LLM could replace the interpreter.They observed a significant decline in performance, indicating that the improvement wasn't solely due to the new prompt style. They also found that meaningful variable names are important, leading to better performance than using random names, as you might expect. 

Demonstartion example:

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



# Multi-step Chain-of-Thoughts
Sonnet 3.5 Intro: "Ever tackled a complex problem by breaking it down into smaller, manageable pieces? That's exactly what these clever 'Plan and Solve' approaches do for language models. They're all about teaching AI to think step-by-step, just like we do. Let's explore how these methods are helping LLMs become better problem solvers!"

## Least-to-Most Prompting (ICLR 2023)
Paper: [Least-to-Most Prompting Enables Complex Reasoning in Large Language Models](https://arxiv.org/pdf/2205.10625)
Code: N/A

The authors claim that CoT perform poorly on tasks that require solving problems harder than the exemplars shown in the prompts (easy-to-hard generalization). They propose a two step approach:

1. Decompose the task into subproblems using few-shot examples showing how to decompose
2. Solve every sub-problem using few-shot examples. Every sub-problem prompt includes the response of its prior sub-problems (hence, must be done sequentially)
    
The main limitation is obvious. You need to make multiple LLM calls sequntially. The other limitation is that you need some trail and error to get the prompts working well for your problem and domain. 

The following image from the paper explains this approach best. 

{{<figure src="least-to-prompt.png" alt="least-to-most-prompting">}}

While this apporach makes a lot of sense, it's just like how humans solve complex problems, it's worth noting that their results show that there are two iportant considerations to make: 1. How trivial is to decompose the task; and 2. How many steps are needed to solve the problem. They show for example that on the GSM8K dataset, least-to-prompt significantly improves on CoT only on questions that require at least 5 reasoning steps. It kinds makes sense. Think of the problem of last-letter concatenation. If the inputs has 2-3 works, one pass with CoT is likely to suffice. But for 10 words, the LLM is likely to get lost somehwere in the middle. Decomposing the problem to 10 sub-problems is expensive but likely lead to the right solution. (they actually reported 100% accuracy on this task)
e were 12,000 Muslims. So there were 93,000 - 12,000 = 81,000 more Mormons than Muslims. So the answer is 81,000.


# Plan-and-Solve Prompting (ACL 2023)
Paper: [Plan-and-Solve Prompting: Improving Zero-Shot Chain-of-Thought Reasoning by Large Language Models](arxiv.org/abs/2305.04091)
Code: https://github.com/agi-edgerunners/plan-and-solve-prompting

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



## Appendix

### In-Context Principle Learning from Mistakes (LEAP)

Similar to self-refine, this work uses the LLM to reflect on its own previously generated outputs. However, the goal here is helping automating the process of writing useful instructions in the prompt (“principles”). It's done offline using gold data with the idea that it can generalize to unseen test examples, so no need to generate multiple samples at test time. The high-level LEAP (Learning Principles) approach goes like this:

- Run the LLM in zero-shot CoT fashion multiple times on some questions you have gold answers for
- Ask the LLM to evaluate answers against the gold 
- For each mistake, ask the LLM to generate low-level principles
- Generate high-level principles from the low-level principles
- Create an enhanced prompt with the low-level and/or high-level principles, with optional few-shot examples (recommended). This is the prompt to be used to label new unseen examples. 

First, the low-level prompt:
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

A low-level principle example:

```
1. The system should be designed to understand the context of the question better. In this case, it should
have recognized that the question was asking for the duration of existence of the European Coal and Steel
Community before it transitioned into the European Economic Community. Understanding the specific
context and requirements of a question is crucial for generating accurate answers
```

Then, optionally, the high-level princpiles prompt:
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

Generated hihg-level princples examples:

```
1. Ensure clarity and precision: Responses should be clear and concise, avoiding any ambiguity or unnecessary complexity.
2. Maintain relevance: Responses should directly address the query or topic at hand, avoiding any unrelated
or tangential information.
1. Prioritize uniqueness: Strive to provide unique insights or perspectives in responses, avoiding repetition or
common knowledge.
```

Final prompt:

```
Instruction: {instruction}
In doing so, please carefully note the following principles:
Principles: {principles}
---
{few_shot_questions_and_answers}
Q: {test_question}
````

The authors show improvements on some datasets over CoT few-shot but not on all. With Llama-2-chat-70B for example, they couldn't see a benefit from the generated principles, even when GPT-4 was used to generate the principles. Also, for the strong models they evaluated (GPT-4, GPT-4 Turbo) the gains were pretty modest for the most part if at all. 

Which principles provide more value?  "We could not identify any particular pattern as to which method should be used. We thus suggest that in real-life scenarios, both approaches should be tested, and selected using a validation set". 

