---
layout: post
title: "Three levels of interpretation"
date: 2020-11-08
---

Lately I've been thinking about how I can avoid over-interpreting the analyses and results in my research. It's so easy to get a over-excited about a result; before you even realize, you start to over-sell your finding (to yourself, to colleagues, etc) and interpret the data beyond what it can actually tell you. You may start to focus on the more speculative aspects of your finding and end up blurring the line between pure speculation and accurate interpretation.
 
In order to avoid this trap, I've started walking through my own analyses with a deliberate exercise that breaks down my interpretation into three tiers. The first two tiers aim to plainly describe the result and interpret it in the context of the experiment and hypothesis. Meanwhile, the third tier is primarily concerned with discussion and sensible speculation. The goal is to a) develop a clear and accurate interpretation of your data, and b) decant any speculation from our description of the data.

I should note here that I think speculation is generally a productive endeavor, which is why it's included in the third tier. We speculate to think of new questions and experiments, or to entertain new ways of thinking about data. Speculation means discussion, which is always welcome. However, it has to be recognized for what it is--mere speculation.

To outline these tiers and how I go about this exercise, let's take a simple bivariate correlation and plot it, and then interpret it:

<img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/scatterplot.png">

### Tier 1

Tier 1 is to simply describe the observed data and report the stats. Here, my notes would look like this:

- *X* and *Y* are positively correlated (r = .865)
- The positive correlation is also statistically significant (p < .001) 

Done. It might also be wise to note any outliers, or other quirks in the data, should they be present. 

### Tier 2

Tier 2 broadens the scope by describing the result in the context of the experiment. Does this finding support your hypothesis—i.e. is it expected? Are these data consistent with the other analyses? Does this finding help predict the results of the analyses you have yet to run? Or, in more exploratory work, does this finding change how you want to approach the next analysis? It might also be useful to note any reason why these results could be incorrect. 

For the sake of this example, my hypothetical notes could look something like this (with references to hypothetical analyses/hypotheses):
- This significant positive correlation supports hypothesis 1, but can't really give any insight into hypothesis 2
- These data make sense with the earlier analysis *A*, and may explain why analysis *B* turned out the way it did
- Based on these results, analysis *C* should look like *\*insert reasonable guess here\**  

Essentially, Tier 1 and 2 should make up the ingredients of a Results section; you want to report your finding and sensibly interpret in the context of your hypotheses and the results of other analyses.   

### Tier 3

Tier 3 moves beyond the Results section and should be potential talking points of a Discussion section. Are these data consistent with other experiments or, more broadly, the literature? What could be the mechanism that produces such a result? What are some confounds that you couldn't control for in your experiment? What alternative explanations are there and what is the best one? Et cetera. Discuss and speculate as much as you want in your Tier 3 notes.

Hypothetical notes could look like this:
- Although we only have a correlation, it might be reasonable to *speculate* on a causal relationship between *X* and *Y* because paper *A* and *B* have identified a mechanism that nicely explain these data 
- Paper *C* has also found similar results in a proper causal paradigm
- However, we have to be careful because our data cannot speak to *\*insert spooky confound here\**
- These data seem to be consistent with *\*major theory\**, but also a somewhat consistent with *\*other major theory\** . At this point we can't really rule out one or the other

And so on. 

## Summary

That's it. Nothing special, but I've found it to be a helpful method that systemically goes through my own analyses. I typically do this in a Jupyter notebook where I can use use a Markdown cell to jot down notes after an analysis. I can use Tier 1 and 2 to design my Results sections, and select appropriate Tier 3 ideas for my Discussion sections. Or, I can use Tier 3 notes for later analysis ideas or experiments altogether. At the end of the day, I now have a nice way to identify when I'm speaking beyond what the data can tell me when I'm excited about a result.  