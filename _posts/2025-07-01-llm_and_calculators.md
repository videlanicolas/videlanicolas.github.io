# LLMs are not calculators

It's been 3 years of AI hype now, and although people now understand some of their limits I still see people misunderstanding what this technology can do. When people talk about AI they really mean LLMs (and generative AI), which is very interesting and fascinating on it's own, both to learn how they work and use as a tool without knowing how it works.

LLMs are trained to guess the most probable outcome to a given piece of text, in its core is thousands of "words" (they are pieces of words called "tokens", but we can call them words, it doesn't really matter for this post), each with a multidimensional (many thousands) vector representation (repeated for homonyms). Each vector gets added or substracted based on what a recurrent neural network is configured to do, and it spits an output that we place a meaning to it. In all this process there was never a moment where the LLM checked that the answer is correct, mainly because it's a hard problem to solve for human written text. Some answeres are easy, like "what's the capital of France?", where the answer is empirical and globally accepted. But one could ask "what happens if you break a mirror?", and the LLM (trained on human written text) might answer "7 years of bad luck", which is not true.

Compare LLMs with calculators. In a calculator I can type "7 x 9" and it'll spit out the result: "63". We don't second guess this result, we don't take a piece of paper and manually verify that "7x9=63", we trust the result blindly. We do so because the calculator was made for this job, to operate numbers, to do math for us. Now ask an LLM the same question. It'll probably return the result "63" as well, since it might have learned that the letters "7", "x", "9", "=" match probably to "63". But we're not sure, this wasn't a machine created to do math, it's a LLM trained to give the most probable result. It's _guessing_, not _calculating_. **I can delegate a complex calculation to a calculator, but I need to double check the work of AIs**.

AI enthusiasts might say "well I could use a neural network to detect when the question is a math calculation, and use a calculator for the asnwer", but that would defeat the idea of LLMs. Unless I'm being very straight and clear "Calculate this: 7 x 9", then figuring out if I need a math operation solved for me can become very challenging. Not to mention this would be a bottelneck for the user waiting for an answer.

LLMs are not calculators, and so they shouldn't be considered one.

## Why this matters?

I see many companies slashing jobs in the name of AI, "hoping" (because it's hope) that eventually AI will do the same job that their employees are currently doing. They are repeating the same behaviour as past companies in the 90s, when the dotcom bubble burst.

The 90s saw the birth of the Internet, a glorious piece of technology that allowed "instant" communication everywhere in the world. People talked how jobs like Real Estate agency and car salesman would be meaningless, since people would just use the Internet to connect with each other. Grocery stores would become warehouses that solely deliver groceries from online purchases, and travel agencies would no longer exist. Fast forward to 2025 and we can see this was not true, we still have all those things. The Internet didn't replace them, it allowed them to grow and scale in size dramatically. Those who didn't understand this were doomed to fail (Blockbuster bankrupt, Walmart having a "Kodak moment").

Leaving out the obviously hard to replace jobs (healthcare, education, food production, legal counseling, etc.), when people talk about "AI taking our jobs" they are talking about office jobs where the computer is the main interface. **LLMs can't replace employees**, because of what we stated before, _LLMs are not calculators_. We constantly need to double check their work, and can't delegate work to them without worrying they'll mess up. At least humans can reason, learn, adapt to the unknown, LLMs have no understanding of the things they spit out. There's no meaning, there's no feedback, they can't thrive and adapt. AI enthusiasts would argue that they can and sometimes better than humans, I would reply with "that's just math, there's nothing else", we can't reduce reasoning to mere 1s and 0s. The fact that is so compelx that its creators can't understand it doesn't mean we achieved some sort of state of conciousness that can learn and reason like humans do.

This is incredibly hard to get some people to understand, I don't know if it's unwillingness to learn how it actually works or people lying about it in order to make the stock market go up. Either way it's a dangerous argument to make (that is, LLMs can replace human jobs). While we did saw "calculists" jobs being replaced by calculators, we're not going to see jobs (such as programmers) being replaced by AI.

# Where it does make sense to use them

LLMs are tools, they are a (very expensive) toy people use on a (as of today) day-to-day basis. It can augment work, primarily one done in a computer. It's incredibly helpful when learning new things, it can easily point to the right path to follow and where to read more details.

Programmers are the ones to be more benefited from it, since now it's more easier than ever to navigate complex ecosystems from scratch (e.g. Android apps). I can see peogrammers asking an AI to throw a basic template for a given app, the boilerplate code, the dependencies that might be useful, some well-known bugs, some initial documentation, etc. It's now very easy to get something started, something programmers can then easily follow on their own, sometimes asking LLMs very precise questions were there's no straightforward answer for it.

I don't see programmers being replaced with AIs, they (AIs) depend on the human knowledge to continue learning. When new CPU architectures comes along, when new libraries are created, when new languages and hardware are used, AIs are going to need humans in order to learn the correct way. AI enthusiasts might say "we can have the AI learng from itself, create a feedback loop from its output towards its input". This is called "synthetic data" and is avoided by all the big companies that have LLMs as products. Synthetic data is bad quality, it's a copy of a copy, slightly degraded with nothing new to learn from it.

# But company X just slashed jobs and replaced them with AI

Then that company either misunderstood what these LLMs really do, or is lying about replacing them with AI.
