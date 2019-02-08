---
layout: post
title: "Working with the Lucene / Solr TokenStream API"
subheading: "Tips and pitfalls from my personal experience."
hero_image_url: "/img/token-streams/hero-whiteboard.jpg"
hero_image_title: ""
hero_image_caption: ""
author: dan
read_time_minutes: 7
tags: [Lucene, Solr, search, tokenstream, tokenizer, tokenfilter]
---

I was recently tasked with a project for a client involving Lucene / Solr
tokenizers and token streams. It's common for most Solr users to only have to
deal with tokenizers at a high level, usually when they're building out
their schema and deciding how their searchable content needs to be broken up
and filtered. For example, adding a standard or whitespace tokenizer, a synonym filter, and a stemmer to the schema. However, the details of this particular project required a deeper level
of involvement. During my investigation on how these things work at a lower
level, I found there was very little documentation available. In fact, the only
documentation I was able to find aside from the javadocs was a blog post by [Michael McCandless](http://blog.mikemccandless.com/2012/04/lucenes-tokenstreams-are-actually.html)
that is almost seven years old! For those people wanting to use the API at a lower level, perhaps to write your own TokenFilter or do some custom tokenization, this isn't much to go on, so I thought I'd do a bit of an update as well as write about some stumbling blocks I came across. But first, let's do a quick API introduction.


### TokenStreams and Tokenizers
TokenStream is the base class that represents a string of text that is intended to either be indexed
or used as a query. Specifically, a TokenStream represents the enumeration of
the _tokens_ of the string of text. That is, it represents the components of
the text after it's been broken into suitable pieces based on your use case (tokenized).
For simple written human text, this typically means splitting on whitespace
and punctuation, and typically 'throwing away' punctuation in a way that
allows the user (the person writing the search query) to not have to type and
match the exact text. You can think of tokenization as a way to normalize and filter both what's being stored in the index and the search query in a way that enables matching from query to index. For example, imagine one of the documents in the index
contains the sentence:
 > It's a sure fire formula for success.

If this sentence were indexed without tokenization, then a user searching for "sure-fire" would not find this document, because that text does not strictly match the indexed sentence.
That's basic tokenization. So what part does a TokenStream _actually_ play? Well, as Mike McCandless explained in his blog post, you can think of TokenStreams in Lucene as a graph where each node is a position and each arc is a token. 

{% include lb-image.html src="/img/token-streams/token-stream2.png" title="TokenStream as a graph" caption="A white-space tokenized string with synonyms" %}

The underlying structure of the code isn't always a graph in the sense that you've got one node pointing to another like you'd traditionally think of a graph, but it can be helpful to think about it that way. TokenStream has Attributes that store all of the information you're interested in about each token. When a TokenStream is to be consumed, `incrementToken()` is called repeatedly as part of the TokenStream lifecycle. TokenStram's job is then to setup the attributes for the token it's currently iterating over. For example, one of the goals of the system is to be able to keep track of positions in the original text while manipulating the underlying representation. A good example is what happens when you want to search using synonyms. In the example sentence above, we added 'sure-fire', 'method', and 'winning' as synonyms. When a user searches for 'sure fire method', they still find the document and the hit highlighting still properly highlights 'sure fire formula' even though the phrases are different lengths.


{% include lb-image.html src="/img/token-streams/tokenstream-hierarchy.png" title="TokenStream class diagram" caption="TokenStream class diagram" %}

There are basically two types of TokenStreams: Tokenizer and TokenFilter.  
A Tokenizer is a TokenStream that takes a Reader as input. Tokenizers are used to transform an input text into a TokenStream through a Reader. This mainly happens as the first step to indexing a piece of content. In other words, the analyzer (starting point) will initially create a TokenStream using a Tokenizer and the following steps will transform it with TokenFilters.  
TokenFilter takes an existing TokenStream and manipulates or filters it in some way. There are lots of TokenFilters already implemented in the Lucene API. Synonyms in Lucene-Solr are implemented as a TokenFilter, essentially adding new arcs to the graph for each synonym.  

### The TokenStream Lifecycle
When TokenStreams are consumed, there's a lifecycle they follow: `reset()`  → `incrementToken()` → `end()` → `close()`. The `incrementToken()` method is where the consumer actually consumes one token at a time, repeatedly, until there are no more tokens to consume. But first, the consumer must `reset()` the input TokenStream.

{% highlight java %}
  /**
   * This method is called by a consumer before it begins consumption using
   * {@link #incrementToken()}.
   * <p>
   * Resets this stream to a clean state. Stateful implementations must implement
   * this method so that they can be reused, just as if they had been created fresh.
   * <p>
   * If you override this method, always call {@code super.reset()}, otherwise
   * some internal state will not be correctly reset (e.g., {@link Tokenizer} will
   * throw {@link IllegalStateException} on further usage).
   */
  public void reset() throws IOException {}
{% endhighlight %}
<div class="caption">Tokenizer.java - <a
href="https://github.com/apache/lucene-solr/blob/branch_7_6/lucene/core/src/java/org/apache/lucene/analysis/TokenStream.java">Solr
7.6 Github</a></div>

> Resets this stream to a clean state. Stateful implementations must implement this method so that they can be reused, just as if they had been created fresh

This is the important bit of that code snippet. This allows you to consume some number of tokens and then start at the beginning again for further processing. It also allows further steps in the chain to reprocess a TokenStream that's already been processed. After consuming all tokens, the consumer will `end()` and then `close()` the TokenStream. It is then ready to move on to the next step where additional consumers may repeat the process on the newly created stream.

### Beware the Tokenizer Lifecycle!
Here's our first pitfall. In the project I was working on, I was using the relatively new ConcatenatingTokenStream class to assemble a few streams together. I initially tried creating a new stream using an existing Tokenizer class and then feeding that into ConcatenatingTokenStream, but I couldn't make it work. I kept running into IllegalStateException telling me that I was violating the TokenStream (lifecycle) contract. After a bit of digging, I found out what was going on. It turns out that Tokenizer implementations themselves violate the contract. In fact, as it stands today, Tokenizer implementations can go through their lifecycle exactly once. No Tokenizer implementation can be `reset()` more than one time. The situation ended up this way because currently Tokenizers are only used in the first step of the analyzer chain. They're then immediately turned into a String based representation that can be `reset()` properly. And, as an optimization, Tokenizer closes and throws away the underlying Reader in order to save memory.

{% highlight java %}
  /**
   * {@inheritDoc}
   * <p>
   * <b>NOTE:</b>
   * The default implementation closes the input Reader, so
   * be sure to call <code>super.close()</code> when overriding this method.
   */
  @Override
  public void close() throws IOException {
    input.close();
    // LUCENE-2387: don't hold onto Reader after close, so
    // GC can reclaim
    inputPending = input = ILLEGAL_STATE_READER;
  }
{% endhighlight %}
<div class="caption">Tokenizer.java - <a
href="https://github.com/apache/lucene-solr/blob/branch_7_6/lucene/core/src/java/org/apache/lucene/analysis/Tokenizer.java">Solr
7.6 Github</a></div>

It also happens that the current Tokenizer implementation doesn't allow overriding in a way that allows subclasses to sidestep the problem if there's a need to use them in another context. What that means for API users is that you cannot use Tokenizers in any place other than the first step of the analyzer chain, and you cannot use a Tokenizer to construct a ConcatenatingTokenStream.

I personally feel this speaks to a deeper API problem. On the one hand, I understand throwing away the Reader. Readers themselves sometimes can't be reset, so it'd be pointless to hold onto them after the initial use. On the other hand, TokenStream specifically states that subclasses must be able to be `reset()`. Tokenizer's inability to do that feels broken and is confusing for the API user. Perhaps Tokenizers should clean up their Reader but store the underlying tokens or strings to be used during the next lifecycle? After all, if the consumer isn't going to process the stream again, then it should throw away the reference so the GC can clean up the whole stream. This would allow all TokenStreams to follow the same lifecycle and clear up the confusion around the contract.

Enough of my opinion though, the point of this post is to help you understand how to use the API in your own projects. So, you're writing a new TokenFilter/TokenFilterFactory, you need to be able to create new a TokenStream on-the-fly, and it needs to be constructed from some kind of String based source. In today's API (Solr 7.6) your only choice is to use a Tokenizer. There is no public TokenStream that can be constructed from a String in the current API. But, Tokenizer doesn't work in this situation. In my project, I had to build a new TokenStream that takes a String as input. Solr 8 will likely contain a new TokenStream class that does take a String as input.

The main thing to beware of is that in the current implementation of the Lucene-Solr API (7.6), you can't feed a Tokenizer to just any interface that takes a TokenStream, because somewhere down the line a consumer may try to `reset()` it and Tokenizer can't be reset.

### Concatenating Token Stream
As I previously mentioned, I'd been working with the new ConcatenatingTokenStream class in my project. The second pitfall I came across is that ConcatenatingTokenStream is broken in Solr 7.6. It doesn't restore its offsets when `end()` is called, and it doesn't `reset()` properly. If you try to use it in Solr 7.6 (or earlier), you will very likely run into problems. Also, the problems can be hard to spot. In some cases, there are no errors thrown. Instead, your indexed content just becomes unsearchable. In other cases, specifically when you've got a copyField in use, you'll run into an IllegalArgumentException talking about startOffset and endOffset being wrong. I did submit a [patch](https://issues.apache.org/jira/browse/LUCENE-8650) to fix the problems which was merged into Solr 7.7.


### Happy Coding
I hope this post gives those of you wanting to use the TokenStream api a bit of a better understanding of how it all comes together, and helps you sidestep some potholes. Comments and feedback are always welcome.