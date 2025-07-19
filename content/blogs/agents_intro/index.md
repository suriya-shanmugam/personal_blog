+++
title = 'What are AI Agents ?'
draft = false
+++

AI agents are becoming a big deal in the world of technology. What makes them special?
Unlike traditional softwares, These are programs that aren’t stuck following one fixed path—they’re built to be flexible. Instead of hard coding what steps to take, they figure things out as they go, depending on the context.

## How is this possible?
This flexibility comes from big improvements in how AI models predict the “next word” (or token) in a sentence, based on what’s already been said. That might sound simple, but it’s opened up a whole new world where these models can make surprisingly smart decisions.

At the heart of these AI agents is something called a large language model (LLM). You can think of it as the agent’s brain. It helps the agent understand what’s going on, figure out what tools it has, and decide what steps to take—without needing someone to guide it every time. In this blog, we’ll explore how these agents work, what makes them special, and why they’re changing the game.

## How AI Agents Use Language Models ?

At the core of every smart AI agent is something called a large language model (LLM)—basically, a very powerful autocomplete engine. LLMs don’t think like humans, but they’re really good at guessing the next word (or piece of a word), one tiny step at a time. These tiny pieces are called tokens, and predicting them well is what helps an agent seem smart or helpful in a conversation.

To keep track of what’s happening in a chat or task, LLMs use something called attention. This lets them “remember” important parts of what’s been said, so they can stay on topic and respond in a useful way. It’s like paying attention to the right parts of a conversation to make a good reply.

We talk to LLMs using something called prompting, which means giving them input in a special format they understand. These prompts include all the roles and messages in the conversation—like who’s the user, who’s the assistant, and what’s already been said. For example:

```py
conversation = [
    {"role": "user", "content": "I need help with my order"},
    {"role": "assistant", "content": "I'd be happy to help. Could you provide your order number?"},
    {"role": "user", "content": "It's ORDER-123"},
]
```
Under the hood, it might look more like this:
```sql
<|im_start|>system
You are a helpful AI assistant named SmolLM, trained by Hugging Face<|im_end|>
<|im_start|>user
I need help with my order<|im_end|>
```
These special tokens (like <|im_start|>) help the model understand who’s speaking and what its role is. By feeding in a prompt like this, the model can decide how to respond—and that’s how agents “think” and talk.

## What Are Tools (and Why Do Agents Need Them)?

While LLMs are great at understanding and generating text, they’re not perfect at everything. Sometimes, they need help—especially when it comes to doing things outside their core skill set, like solving math problems, checking live data, or interacting with other systems. That’s where tools come in.

A tool is basically an extra ability that you give the AI. Think of it like handing a calculator to someone who’s good with words but not great with numbers. The model still drives the conversation, but when it realizes it needs to calculate something, it can call on the calculator tool to do it right.

Another big reason tools are important is that LLMs only know what they were trained on. That means they don’t know what happened in the world after their training finished. So, if your agent needs the current weather, stock prices, or news headlines, you’ll need to give it a tool that can look those things up.

To make this work, we also need to explain each tool to the LLM ahead of time—kind of like giving it instructions. For example, you might say: “Here’s a weather tool. Use it when someone asks about the current weather.” Once the model knows what the tool is for, it will decide on its own whether to use it based on the situation.

So in short: tools make agents smarter, more useful, and more connected to the real world.

## How Do Agents Think and Make Decisions?

To act on their own, AI agents need a way to break down problems, take steps, and figure out when they’ve reached a goal. One popular prompting technique for this is called the Thought-Action-Observation (TAO) cycle. It guides the agent to follow a loop: think about the situation, take an action (like calling a tool), observe what happened, and then do it again—until the task is complete.

If you’ve done any programming, it’s kind of like a while loop
`while (task not complete) → think → act → observe → repeat`

Here’s how a prompt might change during each step of the cycle:

```
User: What’s the weather in New York and do I need an umbrella?

<THINK> I need to get the current weather in New York to answer this. </THINK>
<ACTION> call_weather_api(location="New York") </ACTION>
<OBSERVATION> The forecast says rain with 80% chance. </OBSERVATION>

<THINK> Since it's going to rain, the user will likely need an umbrella. </THINK>
<FINAL_ANSWER> Yes, you should take an umbrella. </FINAL_ANSWER>
```
This cycle helps the agent slow down, reason through the task, and use tools as needed—without jumping straight to a guess.

More recently, some advanced models like DeepSeek-V2 or OpenAI’s o1 are taking this even further. They’ve been fine-tuned to think before answering, using structured tags like <think> and </think> to explicitly separate the reasoning phase from the final response. Unlike TAO, which is just a clever way to craft the prompt, this is a training-level technique, where the model actually learns to reason step by step during training.

So in short: TAO is a prompting method that helps any LLM think more like an agent. But newer models are starting to learn that behavior right out of the box.


## Wrapping Up

Building AI agents might sound magical, but under the hood, there’s a lot going on. You need to design smart prompts, manage the agent’s state as it moves through tasks, handle tool calls, and keep everything in sync across multiple steps. It can get messy fast.

Thankfully, you don’t have to do all of this by hand. Frameworks like SmolAgents and LangGraph take care of much of the heavy lifting. They help you build structured, reliable agents without reinventing the wheel every time.

We’ll explore how these frameworks work—and how you can use them to build your own AI agents—in the next blog. Stay tuned!

References:<br>
[Intro to agents by Hugging face](https://huggingface.co/learn/agents-course/en/unit1/tutorial)<br>
[Deep Learning course by 3Blue1brown](https://www.youtube.com/playlist?list=PLZHQObOWTQDNU6R1_67000Dx_ZCJB-3pi)