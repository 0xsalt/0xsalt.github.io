---
title: "Train AI To Write In My Voice"
date: 2025-07-05 14:02:00 +0000
categories: [AI]
tags: [AI, MLOps]
---

**I didn't realize that to train AI to write more like me, I had to write more, like me.**  

That wasn’t my plan. My plan was to train a model on my writing, let it carry more of the blog post writing load and spit out quality content faster in my voice. My intuition on how AI trains on language, however, was incorrect. Somewhere between the export scripts, markdown filters, and labeled conversation histories, I ended up writing more, not less. 

The irony is not lost on me.

**The Original Problem**: Teach an AI to write in my voice so I can write less, increase my productivity, share more in higher quality at greater speed. Going for the fast, good, AND cheap mythical triad. I'll have this solved in a hot minute, I resolved. 
- Export all of my ChatGPT chats going back 6 months: Check!
- Review chats: Check! 
- Watching my progression from using generic plain chats to focused Project chats to specialized Custom GPT chats over time. When I organized my brain around topics and projects I saw themes developing. Wonderful. 
  Now how do I parse this ... 

- I wrote the first feature for `extract_gpt_conversations.py` script to export based on those items: Plain, Project, GPT. Script: `extract_gpt_conversations.py`
[Github: extract_gpt_conversations.py](https://github.com/0xsalt/chatgpt_conversation_extractor/)

```
./extract_gpt_conversations.py conversations.json

Select category to browse:
0: Project
1: GPT
2: Plain
```

- Now I had a thematically organized archive of my GPT chats, hundreds of conversations going back several months, but full two-sided conversations, lots of "voice" was not mine.
- I configured it to exclude the AI responses in the output because that's not the data / voice style I want to train on.
- I added functions to search forward and backward through ranges and the ability to search chats based on keywords.

```
./extract_gpt_conversations.py conversations.json

Select category to browse:
0: Project
1: GPT
2: Plain
3: List all chats sequentially ASC
4: List all chats sequentially DESC
5: Fuzzy search by keyword

Enter choice: 
```

- Now I uploaded it to OpenAI's API to train a model on this treasure trove of "my voice."
- It accepted a file in JSONL format, a slimmer more compressed version of the JSON file I had available to me. So I wrote the converter for that. Script: `generate_finetune_jsonl.py`
	- [Github: chatgpt_generate_finetune_jsonl.py](https://github.com/0xsalt/chatgpt_generate_finetune_jsonl/)

- Chat content still had too much noise in it so I wrote a script to exclude chats that start with obvious questions indicating lower quality "voice" excerpts. I filtered these out by excluding chats that start with "how," "what," "when," "where," "why," "can," etc. Script: filter_blog_style.py
	- [Github: filter_blog_style.py](https://github.com/0xsalt/chatgpt_generate_finetune_jsonl/)

- Now before I submitted my refined data to train my new custom GPT, I wanted to estimate how much this would cost me. Using Python `tiktoken` library and the current published costs for training GPT-3.5 models
    - price_train_per_1k = 0.012  # Training cost per 1K tokens
    - price_infer_per_1k = 0.008  # Inference cost per 1K tokens
I got to within ~5% variance of the actual cost, around $8 and change. I suspect the difference is most likely internal OpenAI processing steps. Script: `estimate_finetune_cost.py`
	- [Github: estimate_finetune_cost.py](https://github.com/0xsalt/chatgpt_generate_finetune_jsonl/)

Now that I had a data cleanup pipeline in place I could move on to actual model training and use. I worked through semi-automating the file upload and model training job start with OpenAI's API. 

Upload file to stage it
```
openai api files.create -f blog_voice_filtered.jsonl -p fine-tune
```
	-f: Designates the file. Must be in JSONL format.
	You'll get back a file-{XXXXXX} reference designator
	-p: Tells it the purpose. In this case "fine-tune"

List the models available to you
```
openai api models.list
```
- I chose gpt-3.5 for low cost.

List your existing uploaded files. Script: `openai_manage_files.py`
	- [Github: openai_manage_files.py](https://github.com/0xsalt/chatgpt_generate_finetune_jsonl/)
```
./openai_manage_files.py --list
Fetching files from OpenAI API...
Files in OpenAI API:
{
  "id": "file-example123abc456",  // <-- this is what what you need next
  "bytes": 27186,
  "created_at": 1751495630,
  "filename": "step_metrics.csv",
  "object": "file",
  "purpose": "fine-tune-results",
  "status": "uploaded",
  "expires_at": null,
  "status_details": null
}
```

```
openai api fine_tuning.jobs.create -m gpt-3.5-turbo -F file-example123abc456
```
- m: the model ID you want to use
- F: the file ID returned in the --list 

Once I refined my prompt input to stop the model from trying to have a conversation with me and instead output long-form prose, it either replied short and conversational sentences, like it was imitating my side of a conversation, or it just returned my exact original AI generated draft blog post test with no changes. This was not working. It's also where things got interesting. 

**Debug Log: Shifting from Systems Thinking to an MLOps Mindset**
I researched in context of my methodology, data curation, training, and output results and discovered a few things. 
- Situation: Training failure: one-sided conversations of only my voice = useless
- Proposed fix: Provide paired samples of generated blog content + my rewritten styled output. Repeat. 
- Insight: The model needs to train on the transformation action between untrained vs authentic tone. 

I'm moving from a mindset of scripting `while true, if then else, and iterative calculations` to a process of teaching systems how to mirror, reflect, improve.  The model was essentially trying to imitate my side of a phone call because that's the format I provided. I thought it needed to train only on the desired output when what it needed to learn was how to apply the transformation action from one style to another. As I learned to train the model, it was also training me. 

**RLHF: Reinforcement Learning From Human Feedback**
This concept is used as a methodology to train a model to learn what humans prefer, but in this case it was I, the human, that is learning what the model needs in order to behave, to act the way I wanted it to act and output the human desired results. 

You could say I attempted to train a model using RL and HF and, in this case, I wound up being the one getting trained. 

I regularly see that as much as we are tuning and training models, we are also learning how best to interact with them. Many times the most difficult part of automation is explaining our own logic and intentions out loud. Assumptions are discovered quickly when a model does *exactly* what you asked it to and you discover your request had logic flaws. 

It turns out I'm not alone in noticing this shift. A piece in [Ars Technica](https://arstechnica.com/ai/2025/07/how-a-big-shift-in-training-llms-led-to-a-capability-explosion/) published July 7, 2025 highlights how RLHF feedback loops significantly improve model training. Only giving the models all of the right answers isn't enough, it also needs to train on the actions taken to get from various incorrect examples to the desired state. 

**Next Steps: From Imitation to Actual RLHF Feedback Loops**
My upcoming research will focus on providing the model with more examples of before and after writing, and then implementing a proper RLHF-style feedback loop: have it generate two responses, I select the better output most aligned with my goals, and use that as input training examples to its next iteration.

I chuckle at the supposed incongruity of writing more in order to write less, but it starts to make more sense when you realize the model needs to train on the diff of the conversion process, not just the end result. 

I see more writing in my future and eventually, hopefully, less. 

---
