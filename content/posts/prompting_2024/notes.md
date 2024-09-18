
Great tweet: https://x.com/_jasonwei/status/1712551642275655770

Must read: https://reasonwithpal.com

Chain-of-though paper: Chain-of-Thought Prompting Elicits Reasoning in Large Language Models
https://proceedings.neurips.cc/paper_files/paper/2022/file/9d5609613524ecf4f15af0f7b31abca4-Paper-Conference.pdf

"In particular, the widely used chain-of-thought (COT)
method presents the model with the explicit intermediate
steps that are required to reach the final answer. Then, the
model is expected to apply a similar decomposition to the actual test example, and consecutively reach an accurate final
answer"

"Wei et al. (2022) additionally augment each in-context example with chain of thought (COT) intermediate steps. Specifically, each in-context example in the COT setup is a triplet
hxi
, ti
, yii, where xi and yi are input-output pair as before,
and ti
is a natural language description of the steps that are
needed to arrive at the output yi from the input x"

", their performance falls dramatically when dealing with complex arithmetic (Hendrycks et al., 2021; Madaan & Yazdanbakhsh,
2022) or large numbers (Nogueira et al., 2021; Qian et al.,
2022)."

Scratchpads:
Nye, M., Andreassen, A. J., Gur-Ari, G., Michalewski, H.,
Austin, J., Bieber, D., Dohan, D., Lewkowycz, A., Bosma,
M., Luan, D., Sutton, C., and Odena, A. Show your
Work: Scratchpads for Intermediate Computation with
Language Models. arXiv preprint arXiv:2112.00114,
2021.

Self consistency: "Multi-sample Generation As found by Wang et al.
(2022b), chain-of-thought-style methods can be further improved by sampling k > 1 outputs, and selecting the final
answer using majority voting. We thus repeated the greedydecoding experiments using nucleus sampling (Holtzman
et al., 2019) with p = 0.95 and k = 40 as in Lewkowycz
et al. (2022) and temperature of 0.7. As shown in Table 3,
this further increases the accuracy of PAL from 72.0% to
80.4% on GSM8K, obtaining 1.9% higher accuracy than
Minerva-540B using the same number of samples."
Self-Consistency Improves Chain of Thought Reasoning in Language Models.

Here's a summary of a few papers published in the last year or so around prompting I've read recently. This is not an exhustive list by any means, nor a deep dive on any of the papers I'm covering here. 

The general idea of these papers is that they try to improve the commonly used chain-of-thought (CoT) approach (aka let's think step by step), esp on tasks that require more complex reasoning.  One thing that is important to point right away is that the majority of these papers use weaker LLMs than today's top closed models, which means we wouldn't neccasrily be able to reproduce their results with stronger models. But nevertheless, they present simple ideas that can be useful for various use cases and models so it's worth knowing. I will try to provide at least one prompt example for each approach as it's the simplest way to understand the steps. This is why reading paper appendix sections (where all the prompt examples are) of today's NLP paper is sometime more useful that the main sections.

# Let's Try Again Approaches 

In this category we have approaches that generate more than one response at inference time. This is not to be confused with approaches that apply alternatives to the common greedy decoding at inference time (e.g., link). In particular, let's look into Self-Consistency and Self-Refine. 

### Self Refine (Neurips 2023)

Iterative Refinement with Self-Feedback https://proceedings.neurips.cc/paper_files/paper/2023/file/91edff07232fb1b55a505a9e9f6c0ff3-Paper-Conference.pdf
Code: https://github.com/madaan/self-refine

highly cited paper with GPT4 results. Few of the authors are the same. They seem to show better GPT4 improvements than the first paper. 

They key to this strightforward  approach is a Feedback prompt that is guided only by instructions and few-shot examples. There's no model parameter updates like with RL. The method employs three key prompts: one for initial generation, one for feedback, and one for prompt refinement that is developed iterativley. Using a single LLM throughout, the process begins with a standard response generation. The LLM then provides feedback on its own output, which is used to refine the original prompt. This refined prompt generates a new response, with the history of previous iterations incorporated. Essentially, this method automates the iterative process typically performed manually by users.

This is one of the only paper that actually conducted experiments with GPT-4, showing performance gains across datasets, with Math Reasoning showing the least gains. They claim it's due to GPT-4's tendency to overestimate the correctness of its mathematical outputs. They also point notethat the most substantial improvements occur in the early iterations. This is important since runing many iterations is slow and expensive. Another important point is that the feedback prompt is task-specific, tailored to each individual task being performed. This tailoring, while beneficial, may lead to biased results if the prompt becomes overly attuned to the particular errors the LLM tends to make on the test set used.

Let's see an example. One of the tasks in which they observed the highest gains compared to the base models is Constrained Generation, where the model is asked to generate a coherent sentence containing the input concepts given to it. This examples is based on few-shot prompting to guide model outputs, but they show that Self Refine works in zero-shot setups as well. 

Let's say our initial prompt contains a few examplers for the LLM to understand the task:

```
Concepts: ['footage', 'motion', 'ruin', 'tilt', 'window']

Sentence: time lapse footage with tilt up motion of the sun streaking through window of ruin

###

Concepts: ['cause', 'hate', 'hut', 'local', 'love']

Sentence: new beach huts on the island have caused some controversy some locals love them others hate them

###
```

Then the feeback prompt targets generations with missing concepts:

```
Concepts: ['beat', 'drum', 'pen', 'sit', 'use']
Sentence: A man uses a drum to beat a pen.
what concepts from the concept list are missing from the sentence?

Feedback: sit

###

Concepts: ['chair', 'clipper', 'cut', 'hair', 'sit']
Sentence: A girl sitting on the couch with her hair up.
what concepts from the concept list are missing from the sentence?

Feedback: clipper, cut, chair

###

Concepts: ['grass', 'hose', 'spray', 'stand', 'water']
Sentence: A man standing next to a water dripping out of the grass.
what concepts from the concept list are missing from the sentence?

Feedback: hose, spray
```

Now, given a new concept list, the iterative approach could look like the following:

```
###

Concepts: ['card', 'chip', 'deal', 'dealer', 'table']
Sentence: a dealer offers a card to a group of people at a table
what concepts from the concept list are missing from the sentence?

Feedback: chip, deal

Okay, impove the sentence using the sentence:

Sentence: The dealer dealt each person at the table and offered them a card.
what concepts from the concept list are missing from the sentence?

Feedback: chip

Okay, impove the sentence using the sentence:

Sentence: The dealer dealt each person at the table a chip and offered them a card.
what concepts from the concept list are missing from the sentence?

Feedback: NONE

###
```

In this example, the first response was `a dealer offers a card to a group of people at a table` and after a couple of iterations we ended up with an improved version `The dealer dealt each person at the table a chip and offered them a card.`.

### Is the Feedback/Refinment step important?

Can we just generate multiple outputs instead of refining using feedback? The authors ran an experiment to validate this. 
They used the LLM to generate 4 samples (without any feedback and refinment) and asked humans to annotate if a single reponse generated by Self-Refine is better than *all* 4 samples genreated. They reported scores only for two tasks - acronym generation and sentiment reversal. Their results clearly show that humans preferred the Self-Refine generations more often. However, they don't mention how they generated the multiple samples. It seems they used ChatGPT so I'm assuming they just ran it 4 times per input, rather then trying a more common sampling approach. Also, based on Figure 7 in the paper, it seems that ChatGPT with a single reponse performed better than this multi-sample approach on this experiment. Experimental setup matters. 

This experiment is actually a good segway to the next paper. 

Some relevant related work: Reflexion (Shinn et al., 2023) and Self-Correction (Welleck et al., 2022). 


### Self Consistency (ICLR 2023)
SELF-CONSISTENCY IMPROVES CHAIN OF THOUGHT
REASONING IN LANGUAGE MODELS
https://arxiv.org/pdf/2203.11171
Code: 

"We propose a new decoding strategy, self-consistency, to replace the naive greedy decoding used in chain-of-thought prompting."
I like how the authors refer to this approach: "it acts more like a “self-ensemble” that works on top of a single language model." 

We start with a prompt with a set of manually written chain-of-thought exemplars. Then instead of a single response generation using greedy decoding, we generate multiple responsse by using different sampling techiques (e.g., temprature scaling). Finally, we apply majority vote to pick the most “consistent” answer. Note that generations are not identical so they employ a task dependent parser to extract the answer from each generated response. For example, for arithmetic reasoning, they parse the ﬁrst numerical part as the ﬁnal answer after the model generates “The answer is ". This way they can pick the answer that is most common in all generations. 

How they sample? Some combination od temperature sampling (e.g., T = 0.5) and truncated at the top-k (e.g., k = 40) tokens with the highest probability. For GPT-3 they used T = 0.7 without top-k truncation.  They did an ablation study and concluded that self-consistency is generally robust to sampling strategies and parameters.

They show that you can get performance gains with 5-10 samples, but sampling a higher number (e.g., 40) of reasoning paths leads to a consistently better performance. This is the main limitation of this approach as generating 40 outputs is not very practical. 

Importantly, they also showed that self consistency works better than beam search decoding with the same number of beams. Their explanation is that self consistency with sampling generates more diverse samples than beam search. 


Improves performance but too inefficent. 
"We leverage self-consistency (SC) decoding to understand the
upper bound of our method. This sampling-based decoding algorithm can greatly reduce randomness in
the generation procedure and boosts performance. Specifically, we set a temperature of 0.4 and K=40
throughout our experiments"

More examplers usually improve diversity and more consistent outputs. 

# Adaptive Prompting

## Paper: Active Prompting with Chain-of-Thought for Large Language Models (ACL 2024)
### Github: https://github.com/shizhediao/active-prompt

If you wrote a prompt with few-shot CoT you know that coming with examples is not always trivial. You want good examples that will improve the model performance. A common approach is to detect errors that LLM makes and then add those errors to the prompt as examples (a typical ML workflow). This paper automates this step, inspired by active learning in more traditional ML. It applies uncertanity estimation to pick the best examples out of all possible candidates. The approach is really simple - run every example/task though the LLM multiple times and pick those examples where the LLM seems to be inconsistent on with its answer. Then annotate those examples and include them in the prompt. This is nice cause you don't need to have labels in order to find examples where the LLM struggles with, and the extra compute is done "offline" to generate the best prompt, not at inference time like self consistency.

How is uncertanity esimtated? They experiment with Disagreemnt (D) and Entropy (E) based. approaches. 

# https://reasonwithpal.com

Next. PAL: Program-aided Language Models.
Program of Thoughts Prompting: Disentangling Computation from Reasoning for Numerical Reasoning Tasks
Github: https://github.com/TIGER-AI-Lab/Program-of-Thoughts

Two papers with similar ideas that came out about the same time. Let's start with program-of-thoughts.

First, this is cool work. For tasks that require numerical computation, the idea is to let the LLM do the reasoning and language understanding, and delegate the actual computation to a generated program. It makes sense since we know that LLMs are good at generating code and weak at LLMs prone to arithmetic calculation erros, not to mention complex computations.  In my field we tackle math-driven financial questions so this is of personal interest. 

"We evaluate PoT prompting across five MWP datasets, GSM8K, AQuA, SVAMP, TabMWP, MultiArith; and three financial datasets, FinQA, ConvFinQA, and TATQA.". THe usual GSM8K etc but also FinQA which is cool. 

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

Their ablaation study shows that PoT helps more on the more challenging problems in the categories such as ‘linear/polynomial equation’, ‘iterative’, ‘symbolic’, and ‘combinatorics’. But less so on arithmetic for example. 

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
mainly due to the diversity questions in AQuA, which the demonstration cannot possibly cover."

OK, let's see how an example prompt looks like. 

Example:

```
Ques%on: Josh decides to try flipping a house. He buys a house for $80,000 and then puts in $50,000 in repairs.
This increased the value of the house by 150%. How much profit did he make?
# Python code, return ans
cost_of_original_house = 80000
increase_rate = 150 / 100
value_of_house = (1 + increase_rate) * cost_of_original_house
cost_of_repair = 50000
ans = value_of_house - cost_of_repair - cost_of_original_house
```

As you can see, the syntax of their language relis to variables. They talk about it in the paper, why it works better. 

First, let's see how GPT-4o answered this: include image. 

```
To calculate Josh's profit:

1. **Initial Cost**: $80,000 (purchase) + $50,000 (repairs) = $130,000 (total investment).
  
2. **Increased Value**: The house's value increases by 150%, so the new value is:
   \[
   130,000 \times 2.5 = 325,000
   \]
   
3. **Profit**: The profit is the new value minus the total investment:
   \[
   325,000 - 130,000 = 195,000
   \]

Josh made a profit of **$195,000**.
```

Sonnet 3.5 nailed it:

```
Let's approach this problem step-by-step:

1. Calculate the total investment:
   Purchase price + Repairs = $80,000 + $50,000 = $130,000

2. Calculate the new value of the house:
   • The value increased by 150% of the purchase price
   • 150% of $80,000 = $80,000 × 1.5 = $120,000
   • New value = Purchase price + Increase in value
   • New value = $80,000 + $120,000 = $200,000

3. Calculate the profit:
   Profit = New value - Total investment
   Profit = $200,000 - $130,000 = $70,000

Therefore, Josh made a profit of $70,000 on this house flip.
```

I'm not sure why but they have another prompt that has python code:

```
# Write Python Code to solve the following ques7ons. Store your result as a variable named 'ans'.
from sympy import Symbol
from sympy import simplify
import math
from sympy import solve_it
# solve_it(equa7ons, variable): solving the equa7ons and return the variable value.

# Ques7on: In a flight of 600 km, an aircraK was slowed down due to bad weather. Its average speed for
the trip was reduced by 200 km/hr and the 7me of flight increased by 30 minutes. The dura7on of the
flight is:
# Answer op7on: ['A)1 hour', 'B)2 hours', 'C)3 hours', 'D)4 hours', 'E)5 hours']
dura7on = Symbol('dura7on', posi7ve=True)
delay = 30 / 60
total_disntace = 600
original_speed = total_disntace / dura7on
reduced_speed = total_disntace / (dura7on + delay)
solu7on = solve_it(original_speed - reduced_speed - 200, dura7on)
ans = solu7on[dura7on]
```

I think the first prompt is from GSK8 while this one is from the more challenging dataset where they actually used PoT as an intermiddate step. 


The code interpreter tool that you use can do different things. Execute the code, 
Can also use loops etc:

Question: Carlos is planting a lemon tree. The tree will cost $90 to plant. Each year it will grow 7 lemons, which he can sell for $1.5 each. It costs $3 a year to water and feed the tree. How many years will it tak
e before he starts earning money on the lemon tree?
# Python code, return ans
total_cost = 90
cost_of_watering_and_feeding = 3
cost_of_each_lemon = 1.5
num_of_lemon_per_year = 7
ans = 0
while total_cost > 0:
    total_cost += cost_of_watering_and_feeding
    total_cost -= num_of_lemon_per_year * cost_of_each_lemon
    ans += 1

# Question: Find out which of the following values is the multiple of X, if it is divisible by 9 and 12?
# Answer option: ['A)36', 'B)15', 'C)17', 'D)5', 'E)7']
options = [36, 15, 17, 5, 7]
for option in options:
    if option % 9 == 0 and option % 12 == 0:
        ans = option
        break
Some of their prompts have this format:

# Write Python Code to solve the following questions. Store your result as a variable named 'ans'.from sympy import Symbolfrom sympy import simplifyimport mathfrom sympy import solve_it# solve_it(equations, variable): solving the equations and return the variable value.
# Question: In a flight of 600 km, an aircraft was slowed down due to bad weather. Its average speed for the trip was reduced by 200 km/hr and the time of flight increased by 30 minutes. The duration of the flight is:# Answer option: ['A)1 hour', 'B)2 hours', 'C)3 hours', 'D)4 hours', 'E)5 hours']duration = Symbol('duration', positive=True)delay = 30 / 60total_disntace = 600original_speed = total_disntace / durationreduced_speed = total_disntace / (duration + delay)solution = solve_it(original_speed - reduced_speed - 200, duration)ans = solution[duration]
ANother format:

Read the following text and table, and then write code to answer a question:the chart shows that the firm posted market risk 2013related gains on 248 out of 261 days in this period , with 12 days exceeding $ 210 million .december 31 ( in millions ) | 1 basis point increase in jpmorgan chase 2019s credit spread2010 | $ 352009 | $ 39Question: on what percent of trading days were there market gains above $ 210 million?#Pythondays_with_market_gains_above_210_million = 12total_trading_days = 261percent_days_with_market_gains_above_210_million = days_with_market_gains_above_210_million / total_trading_daysans = percent_days_with_market_gains_above_210_million

# BINDING LANGUAGE MODELS IN SYMBOLIC LANGUAGES
https://github.com/xlang-ai/Binder
ICLR 2023
https://arxiv.org/pdf/2210.02875





# PAL

Very similar but generates valid python code. 
"In this paper, we propose Program-Aided Language
model (PAL): a novel method that uses an LLM to read
natural language problems and generate programs as reasoning steps, but offloads the solution step to a Python interpreter"

"This work focuses on COT-style reasoning chain, but in
Appendix I we show that PAL also improves Least-toMost (Zhou et al., 2022) prompts, which introduce reasoning chains that decompose a question into sub-questions"

Nice experiment they did is putting the answers in the prompt to test if the LLM can actually replace the interperter. They found that results dropped signficantly. Which shows the improvement is not just from the new prompt style. Also, they found that meaningful variable names are important than using random names, as you might expect. 

Calculator tool citation: "For example, Cobbe et al. (2021) employ a calculator for arithmetic operations as a post hoc processing". "Unlike these works, PAL generates code
for a Python interpreter, which is general enough to handle
both arithmetic calculations and dates, without specialized
modules and ad-hoc fixes"
Cobbe, K., Kosaraju, V., Bavarian, M., Hilton, J., Nakano,
R., Hesse, C., and Schulman, J. Training Verifiers to Solve Math Word Problems. arXiv preprint
arXiv:2110.14168, 2021.

"Further, LMs that
were pretrained on Python are abundant compared to other
domain-specific languages, making Python code a much
more preferable representation" - they say previous works did semantic parsing where they constrain the decoder
using things like Context-Free Grammar (CFG).

!! " The closest work to ours technically may be Binder (Cheng et al.,
2022), but it addressed mostly answering questions about
tables using SQL and SQL-like Python."

Prompt examples:

```
"""Q: Here is a table where the first line is a header and each subsequent line is a
penguin: name, age, height (cm), weight (kg) Louis, 7, 50, 11 Bernard, 5, 80, 13
Vincent, 9, 60, 11 Gwen, 8, 70, 15 For example: the age of Louis is 7, the weight
of Gwen is 15 kg, the height of Bernard is 80 cm. We now add a penguin to the
table: James, 12, 90, 12
How many penguins are less than 8 years old?
"""
# Put the penguins into a list.
penguins = []
penguins.append(('Louis', 7, 50, 11))
penguins.append(('Bernard', 5, 80, 13))
penguins.append(('Vincent', 9, 60, 11))
penguins.append(('Gwen', 8, 70, 15))
# Add penguin James.
penguins.append(('James', 12, 90, 12))
# Find penguins under 8 years old.
penguins_under_8_years_old = [penguin for penguin in penguins if penguin[1] < 8]
# Count number of perguins under 8.
num_penguin_under_8 = len(penguins_under_8_years_old)
answer = num_penguin_under_8
```

```
# Q: 2015 is coming in 36 hours. What is the date one week from today in MM/DD/YYYY?
# If 2015 is coming in 36 hours, then today is 36 hours before.
today = datetime(2015, 1, 1) - relativedelta(hours=36)
# One week from today,
one_week_from_today = today + relativedelta(weeks=1)
# The answer formatted with %m/%d/%Y is
one_week_from_today.strftime('%m/%d/%Y')
```


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

```
Q: There were nine computers in the server room. Five more computers were installed each day, from monday to thursday. How many computers are now in the server room?

# solution in Python:


def solution():
    """There were nine computers in the server room. Five more computers were installed each day, from monday to thursday. How many computers are now in the server room?"""
    computers_initial = 9
    computers_per_day = 5
    num_days = 4  # 4 days between monday and thursday
    computers_added = computers_per_day * num_days
    computers_total = computers_initial + computers_added
    result = computers_total
    return result
```










# In-Context Principle Learning from Mistakes (LEAP)

This work actually used gpt-4 which is nice. 

This is another nice approach that helps automate the process of writing useful instructions in the prompt (“principles”). We all wrote or saw prompts with many lines of instrctions, DOs and DON'Ts, etc. The high-level LEAP (Learning Principles) approach goes like this:

- Run the LLM in zero-shot CoT fashion multiple times on some questions you have gold answers 
- Ask the LLM to evaluate answers against the gold 
- For each mistake, ask the LLM to generate low-level principles
- Generate high-level principles from the low-level principles
- Create enhanced promp with the low-level and/or high-level principles, with optional few-shot examples (recommended). This is the prompt to be used to label new unseen examples. 

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
