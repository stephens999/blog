---
layout: post
title: Kallisto
---

Having read Lior Pachter's tweets about kallisto, and read his 
[blog post](https://liorpachter.wordpress.com/) 
yesterday, I was excited this morning to sit down and 
read about his new method (with Bray, Pimentel and Melsted), 
kallisto, for "Near-optimal RNA-Seq quantification", just posted on 
[arxiv](http://arxiv.org/pdf/1505.02710v1.pdf)
It's a pretty quick read, at 8 pages + figures, and it didn't take me too long
to understand what was going on, although I did have to use Wikipedia
to look up [path cover](http://en.wikipedia.org/wiki/Path_cover).

Lior's blog post also provides a pretty good summary, so I won't try to
comprehensively 
summarize the paper here: just enough to provide a little context. The bottom
line is that kallisto provides super-fast estimation of transcript abundances
from RNA-seq (less than 5 minutes on a laptop), while (apparently) being almost as accurate as state of the 
art pipelines taking considerable compute resources.
The way it does this is to directly assess, for each read, which
transcripts it is compatible with, by checking the compatibility of
the k-mers in each read (for some suitable k). 
Thus, in contrast
to existing pipelines that first use an alignment algorithm to map
reads to the transcriptome,
kallisto doesn't bother to actually work out
where each read comes from - just assess its compatibility with transcripts. 
It can
get away with this because the compatibility of the read across transcripts
is, essentially, sufficient for estimating the 
relative abundance of the transcripts (see below). 

Another important speed-inducing 
feature is that kallisto only looks for exact matches of k-mers to
the transcriptome. (This is super-fast, essentially 
because you can make an index of all k-mers in the trascriptome.)
So how does it deal with sequencing errors?
Well, most sequencing errors will produce k-mers that are
nowhere in the transcriptome, and these will simply be ignored. The idea is that, provided error rates are low enough (usually are!), 
the remaining k-mers that
don't have errors will be plenty informative enough. 

I'm certainly sold on the speed and convenience. 
Lior makes several interesting points in his blog post about speed.
For example, he notes that the speed is really helpful for software
development: it's much easier to maintain and extend a piece of
software that runs in minutes on a desktop.  

It seems highly likely that this algorithm, and others like it,
will be widely used, and probably supplant the current "pipelines".
So are there any parts of the paper or algorithm that could be improved?

Well, first I don't much care for the likelihood they use for
their quantification (page 7 of their preprint; unfortunately the equations
are un-numbered, but in this case it is the only equation in the text!). 
Let's see if I can render it in markdown here:
$$L(\alpha) = \prod_f \sum_t y_{f,t} \alpha_t.$$

The product here is over fragments (reads) $$f$$, and the sum is over
transcripts $$t$$. Here $$y_{f,t}$$ is an indicator for whether fragment $$f$$
is compatible with transcript $$t$$, and the parameters $$\alpha_t$$ 
(to be estimated) are described as
"the probabilities of selecting fragments from transcripts".

The provenance of this likelihood is unclear to me from their text, but
I think there is a term missing,
 corresponding to the probability of observing a read given
that it was selected from transcript $$t$$. Roughly this should be of order 
$$1/(l_t-r+1)$$ where $$l_t$$ is the length of the transcript $$t$$ and $$r$$ the
 length of the read. 
That is, it should be

$$L(\alpha) = \prod_f \sum_t y_{f,t} \alpha_t (1/(l_t-r+1)).$$

The idea here is that there are $$l_t-r+1$$
possible reads of length $$r$$ that could be emitted by a transcript 
of length $$l_t$$, and if we assume they are all equally likely we get $$1/(l_t-r+1)$$ for each.  
More sophisticated models are certainly possible, but it seems there
should be something here.  Further, it doesn't seem that this would complicate
inference, so it might be interesting to try including it and
see if it explains some of the decrease in accuracy compared with RSEM (Figure 2a) which
I believe does include a term like this.
 
To briefly illustrate why this term could be important, consider two
transcripts, one with exons 1,2, and 3 and another with just exons 1 and 3.
Suppose that, ignoring junction reads that span exon boundaries for a moment,
all reads come from either exon 1 or 3, and none from exon 2. 
This would appear to strongly support expression of the first transcript
and not the second, but their likelihood under these data would
appear to be symmetric in the two transcripts, and not favor
one above the other. Of course the junction reads
would break the symmetry, and the method does make use of these, but
I think this illustrates a potential problem with their likelihood.
 
[Edit: after posting this, the kallisto authors informed me 
that their likelihood without the length term was actually a typo in the 
preprint, and that the length term is implemented in the software. 
Good to know!]

The second comment is rather broader, and it relates to whether 
transcript quantification is even really the right way to proceed. 
The problem is that, when several transcripts (for the same gene say) are very similar, their abundances are not individually identifiable. 
For example, you might be able to say, from the data at a given gene, 
that in total its transcripts are highly abundant, 
but it might be hard to say which ones are abundant. 
In such cases, for many analyses, you might be better off collapsing
the transcript abundances into a single reliably-estimated 
- though perhaps biologically dubious - entity ("gene expression"), 
rather than using the error prone estimates of individual transcript
abundances. One nice feature of the speed of kallisto is that 
it can assess the error in abundance estimates via bootstrapping. In his
blog post Lior 
mentions incorporating this uncertainty into downstream analyses. 
This is certainly an interesting idea, but in extreme cases the
problem is that the measurement error of similar transcripts
may overwhelm any signal in the data. I would be inclined to
also explore the possibility of reducing uncertainty 
by collapsing transcripts. Of course, if the signal you seek is transcript-specific then this may also kill signal. But if you seek signals that are shared among transcripts, attempting quantification and analysis at the level of individual transcripts may be shooting yourself in the foot. 
