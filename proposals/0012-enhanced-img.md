- Start Date: 2018-01-22
- RFC PR: 
- Elix Issues:


# Summary

Build an enhanced version of the <img> tag, the <eix-img> and provide features that mainly improve performance.
The usage will be compatible to the standard HTML element <img>, so it can be a drop-in replacement.
Just use `<elix-img src=whatever.jpg></elix-img>` instead of `<img src=whatever.jpg>`.


# Motivation

The use of <img> might have a serious impact on user experience, we want to improve that where possible.
We want the <elix-img> to improve web page speed and optimize network usage where possible, 
without sacrificing but improving the user experience.

It is not a goal to provide enhanced visual features, like text overlay or alike. ???? Opinions requested !!!!

# Use cases

1. Gallery website
2. Banner carousel
3. Mix of content, e.g. news articles, etc.
4. Side-by-side comparison of native <img /> element, e.g. logo


# Detailed design

Features
1. Lazy-load - Don't download/render anything below the fold
2. Pre-cache - Pre-cache images below the fold ready for when the user needs them (Look at different ways to do this, Fetch, Cache, native async/decode attribute)

????


# Drawbacks

????

# Alternatives

If there is no interestion observer API available just fallback to a native <img /> element.


# Unresolved questions
