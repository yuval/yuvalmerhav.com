+++
title = ''
date = 2024-08-26
draft = true
+++

# Intro

I've been diving into some recent papers on prompt engineering, and I thought I'd share a rundown of what I've found. This isn't a deep dive or anything, just a tour of some interesting stuff from the past year or so. I'm covering works I'm familiar with, mostly from reputable conferences. Not an exhaustive list by any means.

Most of these papers aim to improve upon the standard chain-of-thought (CoT) approach (aks let's think step by step). They particularly focus on tasks that require more complex reasoning and analysis. It's important to note that many of these studies used less advanced language models compared to the current top-tier options like GPT-4o and Claude 3.5 Sonnet. This is why I focus more on the techniques and less on the results. I'm just including an overall results section at the end. 

I'll include at least one prompt example for each approach, as it's often the clearest way to understand the concept. Sometimes, reading the appendix sections of these NLP papers, where all the examples are, is more informative than the main paper. The cool thing is that these ideas are straightforward and easy to implement. It's the part of NLP that's most accessible.

# From Natural Language to Code

You've probably seen ChatGPT using Python before to solve problems. LLMs aren't great when it comes to number crunching. They often make arithmetic mistakes and have trouble with complex calculations. To work around this, we can let the LLM handle the reasoning and code generation (which it's usually good at), then use the generated program to do the actual computation. This can be to achieve a final answer or as an intermidiate step. 

There's been previous work on pairing LLMs with tools like calculators (reference-link) or getting them to translate text into languages like SQL (reference-link). The two papers we're looking at here focus more specifically on math problems and using Python as the programming language. Python as the representation makes sense since it's one of the most popular languages in the datasets LLMs are trained on. Both papers came at the same time and have very similar ideas. 

## Paper 1: PAL: Program-aided Language Models
Github: 

