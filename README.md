# Pagerank Project

In this project, you will create a simple search engine for the website <https://www.lawfareblog.com>.
This website provides legal analysis on US national security issues.

**Due date:** Sunday, 18 September at midnight

**Computation:**
This project has low computational requirements.
You are not required to complete it on the lambda server (although you are welcome to if you'd like).

## Background

**Data:**

The `data` folder contains two files that store example "web graphs".
The file `small.csv.gz` contains the example graph from the *Deeper Inside Pagerank* paper.
This is a small graph, so we can manually inspect the contents of this file with the following command:
```
$ zcat data/small.csv.gz
source,target
1,2
1,3
3,1
3,2
3,5
4,5
4,6
5,6
5,4
6,4
```

> **Recall:**
> The `cat` terminal command outputs the contents of a file to stdout, and the `zcat` command first decompressed a gzipped file and then outputs the decompressed contents.

As you can see, the graph is stored as a CSV file.
The first line is a header,
and each subsequent line stores a single edge in the graph.
The first row contains the source node of the edge and the second row the target node.
The file is assumed to be sorted alphabetically.

The second data file `lawfareblog.csv.gz` contains the link structure for the lawfare blog.
Let's take a look at the first 10 of these lines:
```
$ zcat data/lawfareblog.csv.gz | head
source,target
www.lawfareblog.com/,www.lawfareblog.com/topic/interrogation
www.lawfareblog.com/,www.lawfareblog.com/upcoming-events
www.lawfareblog.com/,www.lawfareblog.com/
www.lawfareblog.com/,www.lawfareblog.com/our-comments-policy
www.lawfareblog.com/,www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
www.lawfareblog.com/,www.lawfareblog.com/topic/lawfare-research-paper-series
www.lawfareblog.com/,www.lawfareblog.com/topic/book-reviews
www.lawfareblog.com/,www.lawfareblog.com/documents-related-mueller-investigation
www.lawfareblog.com/,www.lawfareblog.com/topic/international-law-loac
```
You can see that in this file, the node names are URLs.
Semantically, each line corresponds to an HTML `<a>` tag that is contained in the source webpage and links to the target webpage.

We can use the following command to count the total number of links in the file:
```
$ zcat data/lawfareblog.csv.gz | wc -l
1610789
```
Since every link corresponds to a non-zero entry in the `P` matrix,
this is also the value of `nnz(P)`.
(Technically, we should subtract 1 from this value since the `wc -l` command also counts the header line, not just the data lines.)

To get the dimensions of `P`, we need to count the total number of nodes in the graph.
The following command achieves this by: decompressing the file, extracting the first column, removing all duplicate lines, then counting the results.
```
$ zcat data/lawfareblog.csv.gz | cut -f1 -d, | uniq | wc -l
25761
```
This matrix is large enough that computing matrix products for dense matrices takes several minutes on a single CPU.
Fortunately, however, the matrix is very sparse.
The following python code computes the fraction of entries in the matrix with non-zero values:
```
>>> 1610788 / (25760**2)
0.0024274297384360172
```
Thus, by using sparse matrix operations, we will be able to speed up the code significantly.

**Code:**

The `pagerank.py` file contains code for loading the graph CSV files and searching through their nodes for key phrases.
For example, you can perform a search for all nodes (i.e. urls) that mention the string `corona` with the following command:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --search_query=corona
```

> **NOTE:**
> It will take about 10 seconds to load and parse the data files.
> All the other computation happens essentially instantly.

Currently, the pagerank of the nodes is not currently being calculated correctly, and so the webpages are returned in an arbitrary order.
Your task in this assignment will be to fix these calculations in order to have the most important results (i.e. highest pagerank results) returned first.

## Task 1: the power method

Implement the `WebGraph.power_method` function in `pagerank.py` for computing the pagerank vector by fixing the `FIXME` annotation.

**Part 1:**

To check that your implementation is working,
you should run the program on the `data/small.csv.gz` graph.
For my implementation, I get the following output.
```
$ python3 pagerank.py --data=data/small.csv.gz --verbose
DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=2.5629e-01
DEBUG:root:i=1 residual=1.1841e-01
DEBUG:root:i=2 residual=7.0701e-02
DEBUG:root:i=3 residual=3.1815e-02
DEBUG:root:i=4 residual=2.0497e-02
DEBUG:root:i=5 residual=1.0108e-02
DEBUG:root:i=6 residual=6.3716e-03
DEBUG:root:i=7 residual=3.4228e-03
DEBUG:root:i=8 residual=2.0879e-03
DEBUG:root:i=9 residual=1.1750e-03
DEBUG:root:i=10 residual=7.0131e-04
DEBUG:root:i=11 residual=4.0321e-04
DEBUG:root:i=12 residual=2.3800e-04
DEBUG:root:i=13 residual=1.3812e-04
DEBUG:root:i=14 residual=8.1083e-05
DEBUG:root:i=15 residual=4.7251e-05
DEBUG:root:i=16 residual=2.7704e-05
DEBUG:root:i=17 residual=1.6164e-05
DEBUG:root:i=18 residual=9.4778e-06
DEBUG:root:i=19 residual=5.5066e-06
DEBUG:root:i=20 residual=3.2042e-06
DEBUG:root:i=21 residual=1.8612e-06
DEBUG:root:i=22 residual=1.1283e-06
DEBUG:root:i=23 residual=6.1907e-07
INFO:root:rank=0 pagerank=6.6270e-01 url=4
INFO:root:rank=1 pagerank=5.2179e-01 url=6
INFO:root:rank=2 pagerank=4.1434e-01 url=5
INFO:root:rank=3 pagerank=2.3175e-01 url=2
INFO:root:rank=4 pagerank=1.8590e-01 url=3
INFO:root:rank=5 pagerank=1.6917e-01 url=1
```
Yours likely won't be identical (due to weird floating point issues), but it should be similar.
In particular, the ranking of the nodes/urls should be the same order.

> **NOTE:**
> The `--verbose` flag causes all of the lines beginning with `DEBUG` to be printed.
> By default, only lines beginning with `INFO` are printed.

**Part 2:**

The `pagerank.py` file has an option `--search_query`, which takes a string as a parameter.
If this argument is used, then the program returns all nodes that match the query string sorted according to their pagerank.
Essentially, this gives us the most important pages related to our query.

Again, you may not get the exact same results as me,
but you should get similar results to the examples I've shown below.
Verify that you do in fact get similar results.

```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='corona'
INFO:root:rank=0 pagerank=1.0038e-03 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=1 pagerank=8.9224e-04 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
INFO:root:rank=2 pagerank=7.0390e-04 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=3 pagerank=6.9153e-04 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=4 pagerank=6.7041e-04 url=www.lawfareblog.com/israeli-emergency-regulations-location-tracking-coronavirus-carriers
INFO:root:rank=5 pagerank=6.6256e-04 url=www.lawfareblog.com/why-congress-conducting-business-usual-face-coronavirus
INFO:root:rank=6 pagerank=6.5046e-04 url=www.lawfareblog.com/congressional-homeland-security-committees-seek-ways-support-state-federal-responses-coronavirus
INFO:root:rank=7 pagerank=6.3620e-04 url=www.lawfareblog.com/paper-hearing-experts-debate-digital-contact-tracing-and-coronavirus-privacy-concerns
INFO:root:rank=8 pagerank=6.1248e-04 url=www.lawfareblog.com/house-subcommittee-voices-concerns-over-us-management-coronavirus
INFO:root:rank=9 pagerank=6.0187e-04 url=www.lawfareblog.com/livestream-house-oversight-committee-holds-hearing-government-coronavirus-response

$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='trump'
INFO:root:rank=0 pagerank=5.7826e-03 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=5.2338e-03 url=www.lawfareblog.com/document-trump-revokes-obama-executive-order-counterterrorism-strike-casualty-reporting
INFO:root:rank=2 pagerank=5.1297e-03 url=www.lawfareblog.com/trump-administrations-worrying-new-policy-israeli-settlements
INFO:root:rank=3 pagerank=4.6599e-03 url=www.lawfareblog.com/dc-circuit-overrules-district-courts-due-process-ruling-qasim-v-trump
INFO:root:rank=4 pagerank=4.5934e-03 url=www.lawfareblog.com/donald-trump-and-politically-weaponized-executive-branch
INFO:root:rank=5 pagerank=4.3071e-03 url=www.lawfareblog.com/how-trumps-approach-middle-east-ignores-past-future-and-human-condition
INFO:root:rank=6 pagerank=4.0935e-03 url=www.lawfareblog.com/why-trump-cant-buy-greenland
INFO:root:rank=7 pagerank=3.7591e-03 url=www.lawfareblog.com/oral-argument-summary-qassim-v-trump
INFO:root:rank=8 pagerank=3.4509e-03 url=www.lawfareblog.com/dc-circuit-court-denies-trump-rehearing-mazars-case
INFO:root:rank=9 pagerank=3.4484e-03 url=www.lawfareblog.com/second-circuit-rules-mazars-must-hand-over-trump-tax-returns-new-york-prosecutors

$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='iran'
INFO:root:rank=0 pagerank=4.5746e-03 url=www.lawfareblog.com/praise-presidents-iran-tweets
INFO:root:rank=1 pagerank=4.4174e-03 url=www.lawfareblog.com/how-us-iran-tensions-could-disrupt-iraqs-fragile-peace
INFO:root:rank=2 pagerank=2.6928e-03 url=www.lawfareblog.com/cyber-command-operational-update-clarifying-june-2019-iran-operation
INFO:root:rank=3 pagerank=1.9391e-03 url=www.lawfareblog.com/aborted-iran-strike-fine-line-between-necessity-and-revenge
INFO:root:rank=4 pagerank=1.5452e-03 url=www.lawfareblog.com/parsing-state-departments-letter-use-force-against-iran
INFO:root:rank=5 pagerank=1.5357e-03 url=www.lawfareblog.com/iranian-hostage-crisis-and-its-effect-american-politics
INFO:root:rank=6 pagerank=1.5258e-03 url=www.lawfareblog.com/announcing-united-states-and-use-force-against-iran-new-lawfare-e-book
INFO:root:rank=7 pagerank=1.4221e-03 url=www.lawfareblog.com/us-names-iranian-revolutionary-guard-terrorist-organization-and-sanctions-international-criminal
INFO:root:rank=8 pagerank=1.1788e-03 url=www.lawfareblog.com/iran-shoots-down-us-drone-domestic-and-international-legal-implications
INFO:root:rank=9 pagerank=1.1463e-03 url=www.lawfareblog.com/israel-iran-syria-clash-and-law-use-force
```

**Part 3:**

The webgraph of lawfareblog.com (i.e. the `P` matrix) naturally contains a lot of structure.
For example, essentially all pages on the domain have links to the root page <https://lawfareblog.com/> and other "non-article" pages like <https://www.lawfareblog.com/topics> and <https://www.lawfareblog.com/subscribe-lawfare>.
These pages therefore have a large pagerank.
We can get a list of the pages with the largest pagerank by running

```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz
INFO:root:rank=0 pagerank=2.8741e-01 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=1 pagerank=2.8741e-01 url=www.lawfareblog.com/masthead
INFO:root:rank=2 pagerank=2.8741e-01 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
INFO:root:rank=3 pagerank=2.8741e-01 url=www.lawfareblog.com/documents-related-mueller-investigation
INFO:root:rank=4 pagerank=2.8741e-01 url=www.lawfareblog.com/topics
INFO:root:rank=5 pagerank=2.8741e-01 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
INFO:root:rank=6 pagerank=2.8741e-01 url=www.lawfareblog.com/snowden-revelations
INFO:root:rank=7 pagerank=2.8741e-01 url=www.lawfareblog.com/support-lawfare
INFO:root:rank=8 pagerank=2.8741e-01 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=9 pagerank=2.8741e-01 url=www.lawfareblog.com/our-comments-policy
```

Most of these pages are not very interesting, however, because they are not articles,
and usually when we are performing a web search, we only want articles.

This raises the question: How can we find the most important articles filtering out the non-article pages?
The answer is to modify the `P` matrix by removing all links to non-article pages.

This raises another question: How do we know if a link is a non-article page?
Unfortunately, this is a hard question to answer with 100% accuracy,
but there are many methods that get us most of the way there.
One easy to implement method is to compute what's called the "in-link ratio" of each node (i.e. the total number of edges with the node as a target divided by the total number of nodes),
and then remove nodes from the search results with too-high of a ratio.
The intuition is that non-article pages often appear in the menu of a webpage, and so have links from almost all of the other webpages;
but article-webpages are unlikely to appear on a menu and so will only have a small number of links from other webpages.
The `--filter_ratio` parameter causes the code to remove all pages that have an in-link ratio larger than the provided value.

Using this option, we can estimate the most important articles on the domain with the following command:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2
INFO:root:rank=0 pagerank=3.4696e-01 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=2.9521e-01 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 pagerank=2.9040e-01 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 pagerank=1.5179e-01 url=www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
INFO:root:rank=4 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1963
INFO:root:rank=5 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1964
INFO:root:rank=6 pagerank=1.5071e-01 url=www.lawfareblog.com/lawfare-podcast-week-was-impeachment
INFO:root:rank=7 pagerank=1.4957e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1962
INFO:root:rank=8 pagerank=1.4367e-01 url=www.lawfareblog.com/cyberlaw-podcast-mistrusting-google
INFO:root:rank=9 pagerank=1.4240e-01 url=www.lawfareblog.com/lawfare-podcast-bonus-edition-gordon-sondland-vs-committee-no-bull
```
Notice that the urls in this list look much more like articles than the urls in the previous list.

When Google calculates their `P` matrix for the web,
they use a similar (but much more complicated) process to modify the `P` matrix in order to reduce spam results.
The exact formula they use is a jealously guarded secret that they update continuously.

In the case above, notice that we have accidentally removed the blog's most popular article (<www.lawfareblog.com/snowden-revelations>).
The blog editors believed that Snowden's revelations about NSA spying are so important that they directly put a link to the article on the menu.
So every single webpage in the domain links to the Snowden article,
and our "anti-spam" `--filter-ratio` argument removed this article from the list.
In general, it is a challenging open problem to remove spam from pagerank results,
and all current solutions rely on careful human tuning and still have lots of false positives and false negatives.

**Part 4:**

Recall from the reading that the runtime of pagerank depends heavily on the eigengap of the `\bar\bar P` matrix,
and that this eigengap is bounded by the alpha parameter.

Run the following four commands:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose 
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --alpha=0.99999
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2 --alpha=0.99999
```
You should notice that the last command takes considerably more iterations to compute the pagerank vector.
(My code takes 685 iterations for this call, and about 10 iterations for all the others.)

This raises the question: Why does the second command (with the `--alpha` option but without the `--filter_ratio`) option not take a long time to run?
The answer is that the `P` graph for <https://www.lawfareblog.com> naturally has a large eigengap and so is fast to compute for all alpha values,
but the modified graph does not have a large eigengap and so requires a small alpha for fast convergence.

Changing the value of alpha also gives us very different pagerank rankings.
For example, 
```
$ python3 pagerank_solution.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2
INFO:root:rank=0 pagerank=3.4696e-01 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=2.9521e-01 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 pagerank=2.9040e-01 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 pagerank=1.5179e-01 url=www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
INFO:root:rank=4 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1963
INFO:root:rank=5 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1964
INFO:root:rank=6 pagerank=1.5071e-01 url=www.lawfareblog.com/lawfare-podcast-week-was-impeachment
INFO:root:rank=7 pagerank=1.4957e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1962
INFO:root:rank=8 pagerank=1.4367e-01 url=www.lawfareblog.com/cyberlaw-podcast-mistrusting-google
INFO:root:rank=9 pagerank=1.4240e-01 url=www.lawfareblog.com/lawfare-podcast-bonus-edition-gordon-sondland-vs-committee-no-bull

$ python3 pagerank_solution.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2 --alpha=0.99999
INFO:root:rank=0 pagerank=7.0149e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=7.0149e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.0552e-01 url=www.lawfareblog.com/cost-using-zero-days
INFO:root:rank=3 pagerank=3.1755e-02 url=www.lawfareblog.com/lawfare-podcast-former-congressman-brian-baird-and-daniel-schuman-how-congress-can-continue-function
INFO:root:rank=4 pagerank=2.2040e-02 url=www.lawfareblog.com/events
INFO:root:rank=5 pagerank=1.6027e-02 url=www.lawfareblog.com/water-wars-increased-us-focus-indo-pacific
INFO:root:rank=6 pagerank=1.6026e-02 url=www.lawfareblog.com/water-wars-drill-maybe-drill
INFO:root:rank=7 pagerank=1.6023e-02 url=www.lawfareblog.com/water-wars-disjointed-operations-south-china-sea
INFO:root:rank=8 pagerank=1.6020e-02 url=www.lawfareblog.com/water-wars-song-oil-and-fire
INFO:root:rank=9 pagerank=1.6020e-02 url=www.lawfareblog.com/water-wars-sinking-feeling-philippine-china-relations
```

Which of these rankings is better is entirely subjective,
and the only way to know if you have the "best" alpha for your application is to try several variations and see what is best.
If large alphas are good for your application, you can see that there is a trade-off between quality answers and algorithmic runtime.
We'll be exploring this trade-off more formally in class over the rest of the semester.

## Task 2: the personalization vector

The most interesting applications of pagerank involve the personalization vector.
Implement the `WebGraph.make_personalization_vector` function so that it outputs a personalization vector tuned for the input query.
The pseudocode for the function is:
```
for each index in the personalization vector:
    get the url for the index (see the _url_to_index function)
    check if the url satisfies the input query (see the url_satisfies_query function)
    if so, set the corresponding index to one
normalize the vector
```

**Part 1:**

The command line argument `--personalization_vector_query` will use the function you created above to augment your search with a custom personalization vector.
If you've implemented the function correctly,
you should get results similar to:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona'
INFO:root:rank=0 pagerank=6.3127e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=6.3124e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.5947e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
INFO:root:rank=3 pagerank=1.2209e-01 url=www.lawfareblog.com/brexit-not-immune-coronavirus
INFO:root:rank=4 pagerank=1.2209e-01 url=www.lawfareblog.com/rational-security-my-corona-edition
INFO:root:rank=5 pagerank=9.3360e-02 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
INFO:root:rank=6 pagerank=9.1920e-02 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=7 pagerank=9.1920e-02 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=8 pagerank=7.7770e-02 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=9 pagerank=7.2888e-02 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
```

Notice that these results are significantly different than when using the `--search_query` option:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --search_query='corona'
INFO:root:rank=0 pagerank=8.1320e-03 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
INFO:root:rank=1 pagerank=7.7908e-03 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=2 pagerank=5.2262e-03 url=www.lawfareblog.com/livestream-house-oversight-committee-holds-hearing-government-coronavirus-response
INFO:root:rank=3 pagerank=3.9584e-03 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=4 pagerank=3.8114e-03 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=5 pagerank=3.3973e-03 url=www.lawfareblog.com/paper-hearing-experts-debate-digital-contact-tracing-and-coronavirus-privacy-concerns
INFO:root:rank=6 pagerank=3.3633e-03 url=www.lawfareblog.com/cyberlaw-podcast-how-israel-fighting-coronavirus
INFO:root:rank=7 pagerank=3.3557e-03 url=www.lawfareblog.com/israeli-emergency-regulations-location-tracking-coronavirus-carriers
INFO:root:rank=8 pagerank=3.2160e-03 url=www.lawfareblog.com/congress-needs-coronavirus-failsafe-its-too-late
INFO:root:rank=9 pagerank=3.1036e-03 url=www.lawfareblog.com/why-congress-conducting-business-usual-face-coronavirus
```

Which results are better?
Again, that depends on what you mean by "better."
With the `--personalization_vector_query` option,
a webpage is important only if other coronavirus webpages also think it's important;
with the `--search_query` option,
a webpage is important if any other webpage thinks it's important.
You'll notice that in the later example, many of the webpages are about Congressional proceedings related to the coronavirus.
From a strictly coronavirus perspective, these are not very important webpages.
But in the broader context of national security, these are very important webpages.

Google engineers spend TONs of time fine-tuning their pagerank personalization vectors to remove spam webpages.
Exactly how they do this is another one of their secrets that they don't publicly talk about.

**Part 2:**

Another use of the `--personalization_vector_query` option is that we can find out what webpages are related to the coronavirus but don't directly mention the coronavirus.
This can be used to map out what types of topics are similar to the coronavirus.

For example, the following query ranks all webpages by their `corona` importance,
but removes webpages mentioning `corona` from the results.
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona' --search_query='-corona'
INFO:root:rank=0 pagerank=6.3127e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=6.3124e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.5947e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
INFO:root:rank=3 pagerank=9.3360e-02 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
INFO:root:rank=4 pagerank=7.0277e-02 url=www.lawfareblog.com/fault-lines-foreign-policy-quarantined
INFO:root:rank=5 pagerank=6.9713e-02 url=www.lawfareblog.com/lawfare-podcast-mom-and-dad-talk-clinical-trials-pandemic
INFO:root:rank=6 pagerank=6.4944e-02 url=www.lawfareblog.com/limits-world-health-organization
INFO:root:rank=7 pagerank=5.9492e-02 url=www.lawfareblog.com/chinatalk-dispatches-shanghai-beijing-and-hong-kong
INFO:root:rank=8 pagerank=5.1245e-02 url=www.lawfareblog.com/us-moves-dismiss-case-against-company-linked-ira-troll-farm
INFO:root:rank=9 pagerank=5.1245e-02 url=www.lawfareblog.com/livestream-house-armed-services-holds-hearing-national-security-challenges-north-and-south-america
```
You can see that there are many urls about concepts that are obviously related like "covid", "clinical trials", and "quarantine",
but this algorithm also finds articles about Chinese propaganda and Trump's policy decisions.
Both of these articles are highly relevant to coronavirus discussions,
but a simple keyword search for corona or related terms would not find these articles.

<!--
**Part 3:**

Select another topic related to national security.
You should experiment with a national security topic other than the coronavirus.
For example, find out what articles are important to the `iran` topic but do not contain the word `iran`.
Your goal should be to discover what topics that www.lawfareblog.com considers to be related to the national security topic you choose.
-->

## Submission

1. Create a new repo on github (not a fork of this repo).

1. Run the following commands, and paste their output into the code blocks below.
   
   Task 1, part 1:
   ```
   $ python3 pagerank.py --data=data/small.csv.gz --verbose
   DEBUG:root:computing indices
   DEBUG:root:compu`ting values
   DEBUG:root:i=0 residual=0.3775096527366024
   DEBUG:root:i=1 residual=0.3134881800014478
   DEBUG:root:i=2 residual=0.27565909724769616
   DEBUG:root:i=3 residual=0.21698091174924125
   DEBUG:root:i=4 residual=0.18984193333701746
   DEBUG:root:i=5 residual=0.15531441679114757
   DEBUG:root:i=6 residual=0.13266255320422052
   DEBUG:root:i=7 residual=0.11062269449131636
   DEBUG:root:i=8 residual=0.09351350493437055
   DEBUG:root:i=9 residual=0.07847178461431606
   DEBUG:root:i=10 residual=0.06611071586043013
   DEBUG:root:i=11 residual=0.05558075542317422
   DEBUG:root:i=12 residual=0.04677916959922868
   DEBUG:root:i=13 residual=0.03934897856121125
   DEBUG:root:i=14 residual=0.033108715344134995
   DEBUG:root:i=15 residual=0.02785385629802436
   DEBUG:root:i=16 residual=0.02343484809190327
   DEBUG:root:i=17 residual=0.019716129011772904
   DEBUG:root:i=18 residual=0.016587846030582772
   DEBUG:root:i=19 residual=0.01395577106878237
   DEBUG:root:i=20 residual=0.011741402313411545
   DEBUG:root:i=21 residual=0.009878361997475441
   DEBUG:root:i=22 residual=0.00831094668141982
   DEBUG:root:i=23 residual=0.00699223080150468
   DEBUG:root:i=24 residual=0.005882760480022223
   DEBUG:root:i=25 residual=0.004949331008106634
   DEBUG:root:i=26 residual=0.004164011046610657
   DEBUG:root:i=27 residual=0.00350329916845373
   DEBUG:root:i=28 residual=0.0029474238195818098
   DEBUG:root:i=29 residual=0.002479750273067035
   DEBUG:root:i=30 residual=0.002086283421809434
   DEBUG:root:i=31 residual=0.001755248723892037
   DEBUG:root:i=32 residual=0.0014767399551662134
   DEBUG:root:i=33 residual=0.0012424227196036071
   DEBUG:root:i=34 residual=0.0010452850612512302
   DEBUG:root:i=35 residual=0.0008794276228328276
   DEBUG:root:i=36 residual=0.0007398871106051279
   DEBUG:root:i=37 residual=0.000622487766011449
   DEBUG:root:i=38 residual=0.0005237164064806963
   DEBUG:root:i=39 residual=0.00044061729295670916
   DEBUG:root:i=40 residual=0.00037070367934361246
   DEBUG:root:i=41 residual=0.0003118833964236578
   DEBUG:root:i=42 residual=0.00026239624359076835
   DEBUG:root:i=43 residual=0.00022076131472742284
   DEBUG:root:i=44 residual=0.00018573268200485643
   DEBUG:root:i=45 residual=0.0001562621114119169
   DEBUG:root:i=46 residual=0.00013146769429104669
   DEBUG:root:i=47 residual=0.00011060745616612562
   DEBUG:root:i=48 residual=9.305715311200096e-05
   DEBUG:root:i=49 residual=7.829159121420921e-05
   DEBUG:root:i=50 residual=6.586891011045705e-05
   DEBUG:root:i=51 residual=5.5417360292504586e-05
   DEBUG:root:i=52 residual=4.662417847675578e-05
   DEBUG:root:i=53 residual=3.922622814717091e-05
   DEBUG:root:i=54 residual=3.300212517557336e-05
   DEBUG:root:i=55 residual=2.7765612889676323e-05
   DEBUG:root:i=56 residual=2.3359988329207057e-05
   DEBUG:root:i=57 residual=1.965341290281545e-05
   DEBUG:root:i=58 residual=1.6534967106530907e-05
   DEBUG:root:i=59 residual=1.3911331253029626e-05
   DEBUG:root:i=60 residual=1.170399287410327e-05
   DEBUG:root:i=61 residual=9.846897228687246e-06
   DEBUG:root:i=62 residual=8.284470611960542e-06
   DEBUG:root:i=63 residual=6.969957308151707e-06
   DEBUG:root:i=64 residual=5.864020423072614e-06
   DEBUG:root:i=65 residual=4.93356472615254e-06
   DEBUG:root:i=66 residual=4.150746269876465e-06
   DEBUG:root:i=67 residual=3.492139162946511e-06
   DEBUG:root:i=68 residual=2.938034545119521e-06
   DEBUG:root:i=69 residual=2.4718508008651284e-06
   DEBUG:root:i=70 residual=2.079637351501762e-06
   DEBUG:root:i=71 residual=1.7496571845935734e-06
   DEBUG:root:i=72 residual=1.4720356220004732e-06
   DEBUG:root:i=73 residual=1.2384648209949077e-06
   DEBUG:root:i=74 residual=1.0419551603679174e-06
   DEBUG:root:i=75 residual=8.76626075808464e-07
   INFO:root:rank=0 pagerank=2.1634e+00 url=4
   INFO:root:rank=1 pagerank=1.6664e+00 url=6
   INFO:root:rank=2 pagerank=1.2402e+00 url=5
   INFO:root:rank=3 pagerank=4.5712e-01 url=2
   INFO:root:rank=4 pagerank=3.5619e-01 url=3
   INFO:root:rank=5 pagerank=3.2078e-01 url=1
   ```

   Task 1, part 2:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='corona'
   INFO:root:rank=0 pagerank=4.5865e-03 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
   INFO:root:rank=1 pagerank=4.0464e-03 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
   INFO:root:rank=2 pagerank=2.6118e-03 url=www.lawfareblog.com/britains-coronavirus-response
   INFO:root:rank=3 pagerank=2.5392e-03 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
   INFO:root:rank=4 pagerank=2.3560e-03 url=www.lawfareblog.com/israeli-emergency-regulations-location-tracking-coronavirus-carriers
   INFO:root:rank=5 pagerank=2.2897e-03 url=www.lawfareblog.com/why-congress-conducting-business-usual-face-coronavirus
   INFO:root:rank=6 pagerank=2.2729e-03 url=www.lawfareblog.com/livestream-house-oversight-committee-holds-hearing-government-coronavirus-response
   INFO:root:rank=7 pagerank=2.2522e-03 url=www.lawfareblog.com/congressional-homeland-security-committees-seek-ways-support-state-federal-responses-coronavirus
   INFO:root:rank=8 pagerank=2.1880e-03 url=www.lawfareblog.com/paper-hearing-experts-debate-digital-contact-tracing-and-coronavirus-privacy-concerns
   INFO:root:rank=9 pagerank=2.0341e-03 url=www.lawfareblog.com/cyberlaw-podcast-how-israel-fighting-coronavirus

   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='trump'
   INFO:root:rank=0 pagerank=6.6253e-02 url=www.lawfareblog.com/donald-trump-and-politically-weaponized-executive-branch
   INFO:root:rank=1 pagerank=6.0200e-02 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
   INFO:root:rank=2 pagerank=3.4972e-02 url=www.lawfareblog.com/trump-administrations-worrying-new-policy-israeli-settlements
   INFO:root:rank=3 pagerank=3.2196e-02 url=www.lawfareblog.com/document-trump-revokes-obama-executive-order-counterterrorism-strike-casualty-reporting
   INFO:root:rank=4 pagerank=3.0974e-02 url=www.lawfareblog.com/dc-circuit-overrules-district-courts-due-process-ruling-qasim-v-trump
   INFO:root:rank=5 pagerank=2.8463e-02 url=www.lawfareblog.com/how-trumps-approach-middle-east-ignores-past-future-and-human-condition
   INFO:root:rank=6 pagerank=2.5255e-02 url=www.lawfareblog.com/why-trump-cant-buy-greenland
   INFO:root:rank=7 pagerank=2.2459e-02 url=www.lawfareblog.com/oral-argument-summary-qassim-v-trump
   INFO:root:rank=8 pagerank=2.1464e-02 url=www.lawfareblog.com/dc-circuit-court-denies-trump-rehearing-mazars-case
   INFO:root:rank=9 pagerank=2.1105e-02 url=www.lawfareblog.com/second-circuit-rules-mazars-must-hand-over-trump-tax-returns-new-york-prosecutors

   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='iran'
   INFO:root:rank=0 pagerank=6.6141e-02 url=www.lawfareblog.com/praise-presidents-iran-tweets
   INFO:root:rank=1 pagerank=2.9202e-02 url=www.lawfareblog.com/how-us-iran-tensions-could-disrupt-iraqs-fragile-peace
   INFO:root:rank=2 pagerank=1.7711e-02 url=www.lawfareblog.com/cyber-command-operational-update-clarifying-june-2019-iran-operation
   INFO:root:rank=3 pagerank=1.4606e-02 url=www.lawfareblog.com/aborted-iran-strike-fine-line-between-necessity-and-revenge
   INFO:root:rank=4 pagerank=8.4519e-03 url=www.lawfareblog.com/iranian-hostage-crisis-and-its-effect-american-politics
   INFO:root:rank=5 pagerank=8.3997e-03 url=www.lawfareblog.com/parsing-state-departments-letter-use-force-against-iran
   INFO:root:rank=6 pagerank=8.2589e-03 url=www.lawfareblog.com/announcing-united-states-and-use-force-against-iran-new-lawfare-e-book
   INFO:root:rank=7 pagerank=8.0568e-03 url=www.lawfareblog.com/trump-moves-cut-irans-oil-revenues-whats-his-endgame
   INFO:root:rank=8 pagerank=7.1946e-03 url=www.lawfareblog.com/us-names-iranian-revolutionary-guard-terrorist-organization-and-sanctions-international-criminal
   INFO:root:rank=9 pagerank=5.9410e-03 url=www.lawfareblog.com/iran-shoots-down-us-drone-domestic-and-international-legal-implications

   ```

   Task 1, part 3:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz
   INFO:root:rank=0 pagerank=8.4175e+00 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
   INFO:root:rank=1 pagerank=8.4175e+00 url=www.lawfareblog.com/lawfare-job-board
   INFO:root:rank=2 pagerank=8.4175e+00 url=www.lawfareblog.com/documents-related-mueller-investigation
   INFO:root:rank=3 pagerank=8.4175e+00 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban
   INFO:root:rank=4 pagerank=8.4175e+00 url=www.lawfareblog.com/subscribe-lawfare
   INFO:root:rank=5 pagerank=8.4175e+00 url=www.lawfareblog.com/masthead
   INFO:root:rank=6 pagerank=8.4175e+00 url=www.lawfareblog.com/topics
   INFO:root:rank=7 pagerank=8.4175e+00 url=www.lawfareblog.com/our-comments-policy
   INFO:root:rank=8 pagerank=8.4175e+00 url=www.lawfareblog.com/upcoming-events
   INFO:root:rank=9 pagerank=8.4175e+00 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site

   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2
   INFO:root:rank=0 pagerank=4.6096e+00 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
   INFO:root:rank=1 pagerank=2.9870e+00 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
   INFO:root:rank=2 pagerank=2.9672e+00 url=www.lawfareblog.com/opening-statement-david-holmes
   INFO:root:rank=3 pagerank=2.0175e+00 url=www.lawfareblog.com/senate-examines-threats-homeland
   INFO:root:rank=4 pagerank=1.8771e+00 url=www.lawfareblog.com/what-make-first-day-impeachment-hearings
   INFO:root:rank=5 pagerank=1.8764e+00 url=www.lawfareblog.com/livestream-house-armed-services-committee-hearing-f-35-program
   INFO:root:rank=6 pagerank=1.8695e+00 url=www.lawfareblog.com/whats-house-resolution-impeachment
   INFO:root:rank=7 pagerank=1.7657e+00 url=www.lawfareblog.com/congress-us-policy-toward-syria-and-turkey-overview-recent-hearings
   INFO:root:rank=8 pagerank=1.6809e+00 url=www.lawfareblog.com/summary-david-holmess-deposition-testimony
   INFO:root:rank=9 pagerank=9.8355e-01 url=www.lawfareblog.com/events

   ```

   Task 1, part 4:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose 
   DEBUG:root:computing indices
   DEBUG:root:computing values
   DEBUG:root:i=0 residual=20.52150126053022
   DEBUG:root:i=1 residual=6.111376245092865
   DEBUG:root:i=2 residual=1.922378236879763
   DEBUG:root:i=3 residual=0.5881268315375173
   DEBUG:root:i=4 residual=0.17579780291491676
   DEBUG:root:i=5 residual=0.05184146773392674
   DEBUG:root:i=6 residual=0.015210253368731745
   DEBUG:root:i=7 residual=0.004471865137492178
   DEBUG:root:i=8 residual=0.001327715838783267
   DEBUG:root:i=9 residual=0.00040385624967524095
   DEBUG:root:i=10 residual=0.0001302219221427986
   DEBUG:root:i=11 residual=4.789799574118051e-05
   DEBUG:root:i=12 residual=2.224957913429871e-05
   DEBUG:root:i=13 residual=1.3517296444959574e-05
   DEBUG:root:i=14 residual=9.912341451294986e-06
   DEBUG:root:i=15 residual=7.960146875305924e-06
   DEBUG:root:i=16 residual=6.627685440395496e-06
   DEBUG:root:i=17 residual=5.5921326791259875e-06
   DEBUG:root:i=18 residual=4.740892614699404e-06
   DEBUG:root:i=19 residual=4.026024158714828e-06
   DEBUG:root:i=20 residual=3.4209953991880717e-06
   DEBUG:root:i=21 residual=2.907506476158979e-06
   DEBUG:root:i=22 residual=2.471277971751155e-06
   DEBUG:root:i=23 residual=2.1005550878288405e-06
   DEBUG:root:i=24 residual=1.7854625121314395e-06
   DEBUG:root:i=25 residual=1.5176402781311293e-06
   DEBUG:root:i=26 residual=1.2899933678749356e-06
   DEBUG:root:i=27 residual=1.096494168281018e-06
   DEBUG:root:i=28 residual=9.320199924343523e-07
   INFO:root:rank=0 pagerank=8.4175e+00 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
   INFO:root:rank=1 pagerank=8.4175e+00 url=www.lawfareblog.com/lawfare-job-board
   INFO:root:rank=2 pagerank=8.4175e+00 url=www.lawfareblog.com/documents-related-mueller-investigation
   INFO:root:rank=3 pagerank=8.4175e+00 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban
   INFO:root:rank=4 pagerank=8.4175e+00 url=www.lawfareblog.com/subscribe-lawfare
   INFO:root:rank=5 pagerank=8.4175e+00 url=www.lawfareblog.com/masthead
   INFO:root:rank=6 pagerank=8.4175e+00 url=www.lawfareblog.com/topics
   INFO:root:rank=7 pagerank=8.4175e+00 url=www.lawfareblog.com/our-comments-policy
   INFO:root:rank=8 pagerank=8.4175e+00 url=www.lawfareblog.com/upcoming-events
   INFO:root:rank=9 pagerank=8.4175e+00 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
   
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --alpha=0.99999
   DEBUG:root:computing indices
   DEBUG:root:computing values
   DEBUG:root:i=0 residual=24.14269911590951
   DEBUG:root:i=1 residual=8.45841801072323
   DEBUG:root:i=2 residual=3.1300902278182847
   DEBUG:root:i=3 residual=1.1265256008835056
   DEBUG:root:i=4 residual=0.39608640229301556
   DEBUG:root:i=5 residual=0.13734970247851327
   DEBUG:root:i=6 residual=0.04734527255534044
   DEBUG:root:i=7 residual=0.016312159820305958
   DEBUG:root:i=8 residual=0.005634492774401902
   DEBUG:root:i=9 residual=0.0019538783765715296
   DEBUG:root:i=10 residual=0.0006806130757695915
   DEBUG:root:i=11 residual=0.00023837136570336085
   DEBUG:root:i=12 residual=8.418570703431892e-05
   DEBUG:root:i=13 residual=3.025204314332667e-05
   DEBUG:root:i=14 residual=1.1336013928676048e-05
   DEBUG:root:i=15 residual=4.692882473452792e-06
   DEBUG:root:i=16 residual=2.3660064653315943e-06
   DEBUG:root:i=17 residual=1.5575735379930745e-06
   DEBUG:root:i=18 residual=1.2777885960996104e-06
   DEBUG:root:i=19 residual=1.180259960270273e-06
   DEBUG:root:i=20 residual=1.145936666971802e-06
   DEBUG:root:i=21 residual=1.1337771421730776e-06
   DEBUG:root:i=22 residual=1.1294485280958716e-06
   DEBUG:root:i=23 residual=1.1278988503036985e-06
   DEBUG:root:i=24 residual=1.1273381562397344e-06
   DEBUG:root:i=25 residual=1.127130239963562e-06
   DEBUG:root:i=26 residual=1.1270484625864706e-06
   DEBUG:root:i=27 residual=1.1270118423989063e-06
   DEBUG:root:i=28 residual=1.1269915013519983e-06
   DEBUG:root:i=29 residual=1.12697697525551e-06
   DEBUG:root:i=30 residual=1.1269645764745394e-06
   DEBUG:root:i=31 residual=1.1269528500336565e-06
   DEBUG:root:i=32 residual=1.1269415240630582e-06
   DEBUG:root:i=33 residual=1.1269302045378558e-06
   DEBUG:root:i=34 residual=1.126918934263866e-06
   DEBUG:root:i=35 residual=1.1269076886412436e-06
   DEBUG:root:i=36 residual=1.1268964368701495e-06
   DEBUG:root:i=37 residual=1.1268851974112095e-06
   DEBUG:root:i=38 residual=1.1268739887138637e-06
   DEBUG:root:i=39 residual=1.1268627369759502e-06
   DEBUG:root:i=40 residual=1.1268515221170058e-06
   DEBUG:root:i=41 residual=1.1268403011349884e-06
   DEBUG:root:i=42 residual=1.1268290370802038e-06
   DEBUG:root:i=43 residual=1.1268178222271258e-06
   DEBUG:root:i=44 residual=1.1268066073860328e-06
   DEBUG:root:i=45 residual=1.1267953740977388e-06
   DEBUG:root:i=46 residual=1.126784146954254e-06
   DEBUG:root:i=47 residual=1.1267729136558655e-06
   DEBUG:root:i=48 residual=1.1267616619022293e-06
   DEBUG:root:i=49 residual=1.1267504162898838e-06
   DEBUG:root:i=50 residual=1.1267391829806964e-06
   DEBUG:root:i=51 residual=1.1267279312294127e-06
   DEBUG:root:i=52 residual=1.1267167225245282e-06
   DEBUG:root:i=53 residual=1.1267055200026875e-06
   DEBUG:root:i=54 residual=1.126694249802188e-06
   DEBUG:root:i=55 residual=1.12668302264228e-06
   DEBUG:root:i=56 residual=1.1266718016448776e-06
   DEBUG:root:i=57 residual=1.1266605683544122e-06
   DEBUG:root:i=58 residual=1.1266493227547694e-06
   DEBUG:root:i=59 residual=1.1266380709907367e-06
   DEBUG:root:i=60 residual=1.1266268684404974e-06
   DEBUG:root:i=61 residual=1.1266156351547938e-06
   DEBUG:root:i=62 residual=1.1266043834060967e-06
   DEBUG:root:i=63 residual=1.1265931500951994e-06
   DEBUG:root:i=64 residual=1.1265819044902635e-06
   DEBUG:root:i=65 residual=1.1265706834873122e-06
   DEBUG:root:i=66 residual=1.1265594317413536e-06
   DEBUG:root:i=67 residual=1.12654823534284e-06
   DEBUG:root:i=68 residual=1.1265369774529053e-06
   DEBUG:root:i=69 residual=1.1265257441442916e-06
   DEBUG:root:i=70 residual=1.1265145662114339e-06
   DEBUG:root:i=71 residual=1.1265033206380836e-06
   DEBUG:root:i=72 residual=1.1264920688795278e-06
   DEBUG:root:i=73 residual=1.1264808540235746e-06
   DEBUG:root:i=74 residual=1.1264696330386035e-06
   DEBUG:root:i=75 residual=1.1264583566781683e-06
   DEBUG:root:i=76 residual=1.1264471479711059e-06
   DEBUG:root:i=77 residual=1.126435908528535e-06
   DEBUG:root:i=78 residual=1.1264246752272718e-06
   DEBUG:root:i=79 residual=1.1264134173218272e-06
   DEBUG:root:i=80 residual=1.126402196311304e-06
   DEBUG:root:i=81 residual=1.1263909691720645e-06
   DEBUG:root:i=82 residual=1.1263797666347654e-06
   DEBUG:root:i=83 residual=1.126368496441748e-06
   DEBUG:root:i=84 residual=1.1263572754261876e-06
   DEBUG:root:i=85 residual=1.1263460544360414e-06
   DEBUG:root:i=86 residual=1.1263347965358397e-06
   DEBUG:root:i=87 residual=1.1263235755305105e-06
   DEBUG:root:i=88 residual=1.1263123422372834e-06
   DEBUG:root:i=89 residual=1.1263011581495338e-06
   DEBUG:root:i=90 residual=1.1262899187234354e-06
   DEBUG:root:i=91 residual=1.1262786669724932e-06
   DEBUG:root:i=92 residual=1.126267409051747e-06
   DEBUG:root:i=93 residual=1.1262562126528656e-06
   DEBUG:root:i=94 residual=1.126245016280242e-06
   DEBUG:root:i=95 residual=1.1262337645477582e-06
   DEBUG:root:i=96 residual=1.1262225189357663e-06
   DEBUG:root:i=97 residual=1.126211297935889e-06
   DEBUG:root:i=98 residual=1.1262000646392523e-06
   DEBUG:root:i=99 residual=1.1261888621023762e-06
   DEBUG:root:i=100 residual=1.12617764727485e-06
   DEBUG:root:i=101 residual=1.1261664016783894e-06
   DEBUG:root:i=102 residual=1.1261551745292926e-06
   DEBUG:root:i=103 residual=1.12614394738506e-06
   DEBUG:root:i=104 residual=1.1261327325419978e-06
   DEBUG:root:i=105 residual=1.1261215115575267e-06
   DEBUG:root:i=106 residual=1.1261102905676541e-06
   DEBUG:root:i=107 residual=1.1260990880360582e-06
   DEBUG:root:i=108 residual=1.1260878239921685e-06
   DEBUG:root:i=109 residual=1.1260766029868435e-06
   DEBUG:root:i=110 residual=1.126065357384938e-06
   DEBUG:root:i=111 residual=1.1260541302356583e-06
   DEBUG:root:i=112 residual=1.1260428907799874e-06
   DEBUG:root:i=113 residual=1.1260316820935436e-06
   DEBUG:root:i=114 residual=1.1260204918651537e-06
   DEBUG:root:i=115 residual=1.1260092954980095e-06
   DEBUG:root:i=116 residual=1.125998031465181e-06
   DEBUG:root:i=117 residual=1.1259868043005829e-06
   DEBUG:root:i=118 residual=1.1259755525490624e-06
   DEBUG:root:i=119 residual=1.125964319240795e-06
   DEBUG:root:i=120 residual=1.1259531536108e-06
   DEBUG:root:i=121 residual=1.1259418834332303e-06
   DEBUG:root:i=122 residual=1.1259306501222113e-06
   DEBUG:root:i=123 residual=1.125919416818602e-06
   DEBUG:root:i=124 residual=1.125908208126771e-06
   DEBUG:root:i=125 residual=1.1258969748439967e-06
   DEBUG:root:i=126 residual=1.1258857661450166e-06
   DEBUG:root:i=127 residual=1.1258745451679984e-06
   DEBUG:root:i=128 residual=1.1258632995717088e-06
   DEBUG:root:i=129 residual=1.1258520724201515e-06
   DEBUG:root:i=130 residual=1.1258408821838404e-06
   DEBUG:root:i=131 residual=1.1258296612119088e-06
   DEBUG:root:i=132 residual=1.1258184094640881e-06
   DEBUG:root:i=133 residual=1.12580715770296e-06
   DEBUG:root:i=134 residual=1.1257959736054026e-06
   DEBUG:root:i=135 residual=1.125784728028842e-06
   DEBUG:root:i=136 residual=1.1257734824221793e-06
   DEBUG:root:i=137 residual=1.125762298327435e-06
   DEBUG:root:i=138 residual=1.1257510589055422e-06
   DEBUG:root:i=139 residual=1.1257398379056513e-06
   DEBUG:root:i=140 residual=1.1257286169160635e-06
   DEBUG:root:i=141 residual=1.1257174082328154e-06
   DEBUG:root:i=142 residual=1.1257061626410532e-06
   DEBUG:root:i=143 residual=1.125694941641492e-06
   DEBUG:root:i=144 residual=1.1256837391065172e-06
   DEBUG:root:i=145 residual=1.1256725181324987e-06
   DEBUG:root:i=146 residual=1.125661297138138e-06
   DEBUG:root:i=147 residual=1.1256500638474527e-06
   DEBUG:root:i=148 residual=1.1256388551590348e-06
   DEBUG:root:i=149 residual=1.125627628022333e-06
   DEBUG:root:i=150 residual=1.1256164254860136e-06
   DEBUG:root:i=151 residual=1.1256051860488222e-06
   DEBUG:root:i=152 residual=1.1255939465984704e-06
   DEBUG:root:i=153 residual=1.1255827686635966e-06
   DEBUG:root:i=154 residual=1.1255715230873204e-06
   DEBUG:root:i=155 residual=1.1255602897819384e-06
   DEBUG:root:i=156 residual=1.1255490626401023e-06
   DEBUG:root:i=157 residual=1.1255378416428989e-06
   DEBUG:root:i=158 residual=1.1255266145035393e-06
   DEBUG:root:i=159 residual=1.1255154119668656e-06
   DEBUG:root:i=160 residual=1.1255041786811972e-06
   DEBUG:root:i=161 residual=1.1254929884505724e-06
   DEBUG:root:i=162 residual=1.125481761322302e-06
   DEBUG:root:i=163 residual=1.1254705218779011e-06
   DEBUG:root:i=164 residual=1.1254593008776746e-06
   DEBUG:root:i=165 residual=1.1254481106489336e-06
   DEBUG:root:i=166 residual=1.1254368589194578e-06
   DEBUG:root:i=167 residual=1.1254256379114057e-06
   DEBUG:root:i=168 residual=1.1254144353793323e-06
   DEBUG:root:i=169 residual=1.1254032020940806e-06
   DEBUG:root:i=170 residual=1.1253919872541363e-06
   DEBUG:root:i=171 residual=1.1253807539686216e-06
   DEBUG:root:i=172 residual=1.125369563722114e-06
   DEBUG:root:i=173 residual=1.125358330455085e-06
   DEBUG:root:i=174 residual=1.1253470786989686e-06
   DEBUG:root:i=175 residual=1.1253359192057388e-06
   DEBUG:root:i=176 residual=1.1253246674963186e-06
   DEBUG:root:i=177 residual=1.1253134526463316e-06
   DEBUG:root:i=178 residual=1.125302182442393e-06
   DEBUG:root:i=179 residual=1.1252910167976046e-06
   DEBUG:root:i=180 residual=1.1252798142837278e-06
   DEBUG:root:i=181 residual=1.125268611767486e-06
   DEBUG:root:i=182 residual=1.1252573477245704e-06
   DEBUG:root:i=183 residual=1.1252461943883313e-06
   DEBUG:root:i=184 residual=1.1252349242155418e-06
   DEBUG:root:i=185 residual=1.1252237032035591e-06
   DEBUG:root:i=186 residual=1.1252125068203483e-06
   DEBUG:root:i=187 residual=1.1252012612365785e-06
   DEBUG:root:i=188 residual=1.1251900463886735e-06
   DEBUG:root:i=189 residual=1.1251788315532265e-06
   DEBUG:root:i=190 residual=1.1251675859619578e-06
   DEBUG:root:i=191 residual=1.1251564141729014e-06
   DEBUG:root:i=192 residual=1.1251451747621941e-06
   DEBUG:root:i=193 residual=1.1251339722152692e-06
   DEBUG:root:i=194 residual=1.1251227389269214e-06
   DEBUG:root:i=195 residual=1.1251115302410556e-06
   DEBUG:root:i=196 residual=1.1251003092565362e-06
   DEBUG:root:i=197 residual=1.1250891190285734e-06
   DEBUG:root:i=198 residual=1.1250778980544703e-06
   DEBUG:root:i=199 residual=1.1250666832198184e-06
   DEBUG:root:i=200 residual=1.125055449931868e-06
   DEBUG:root:i=201 residual=1.1250442350889144e-06
   DEBUG:root:i=202 residual=1.1250330387059876e-06
   DEBUG:root:i=203 residual=1.1250218177349087e-06
   DEBUG:root:i=204 residual=1.1250105967453987e-06
   DEBUG:root:i=205 residual=1.1249993634524498e-06
   DEBUG:root:i=206 residual=1.1249881486094938e-06
   DEBUG:root:i=207 residual=1.1249769891420645e-06
   DEBUG:root:i=208 residual=1.124965737428022e-06
   DEBUG:root:i=209 residual=1.1249545225751828e-06
   DEBUG:root:i=210 residual=1.1249433077367047e-06
   DEBUG:root:i=211 residual=1.1249321052133326e-06
   DEBUG:root:i=212 residual=1.1249208780823561e-06
   DEBUG:root:i=213 residual=1.1249096447818036e-06
   DEBUG:root:i=214 residual=1.1248984422496387e-06
   DEBUG:root:i=215 residual=1.1248872335663575e-06
   DEBUG:root:i=216 residual=1.1248759941370387e-06
   DEBUG:root:i=217 residual=1.1248647608287312e-06
   DEBUG:root:i=218 residual=1.1248535644429749e-06
   DEBUG:root:i=219 residual=1.1248423373146695e-06
   DEBUG:root:i=220 residual=1.124831116325228e-06
   DEBUG:root:i=221 residual=1.1248199076367106e-06
   DEBUG:root:i=222 residual=1.1248087051124497e-06
   DEBUG:root:i=223 residual=1.1247974841334185e-06
   DEBUG:root:i=224 residual=1.1247863000522444e-06
   DEBUG:root:i=225 residual=1.1247750913917923e-06
   DEBUG:root:i=226 residual=1.1247638457985378e-06
   DEBUG:root:i=227 residual=1.1247526248014548e-06
   DEBUG:root:i=228 residual=1.124741459179767e-06
   DEBUG:root:i=229 residual=1.1247302197610505e-06
   DEBUG:root:i=230 residual=1.1247189926123782e-06
   DEBUG:root:i=231 residual=1.1247078085305604e-06
   DEBUG:root:i=232 residual=1.1246965937159574e-06
   DEBUG:root:i=233 residual=1.1246853788838885e-06
   DEBUG:root:i=234 residual=1.1246741455883388e-06
   DEBUG:root:i=235 residual=1.1246629246014328e-06
   DEBUG:root:i=236 residual=1.1246517036090565e-06
   DEBUG:root:i=237 residual=1.1246405318335232e-06
   DEBUG:root:i=238 residual=1.1246292862653705e-06
   DEBUG:root:i=239 residual=1.1246180714178685e-06
   DEBUG:root:i=240 residual=1.1246068750348288e-06
   DEBUG:root:i=241 residual=1.1245956540612895e-06
   DEBUG:root:i=242 residual=1.1245844330741634e-06
   DEBUG:root:i=243 residual=1.124573199781389e-06
   DEBUG:root:i=244 residual=1.1245620156970323e-06
   DEBUG:root:i=245 residual=1.1245508070313999e-06
   DEBUG:root:i=246 residual=1.1245395860475472e-06
   DEBUG:root:i=247 residual=1.124528377366969e-06
   DEBUG:root:i=248 residual=1.1245171994444803e-06
   DEBUG:root:i=249 residual=1.124505978478748e-06
   DEBUG:root:i=250 residual=1.1244947574973681e-06
   DEBUG:root:i=251 residual=1.1244835426546677e-06
   DEBUG:root:i=252 residual=1.1244723278218804e-06
   DEBUG:root:i=253 residual=1.124461137596304e-06
   DEBUG:root:i=254 residual=1.1244499289231186e-06
   DEBUG:root:i=255 residual=1.1244387325515858e-06
   DEBUG:root:i=256 residual=1.1244274992768128e-06
   DEBUG:root:i=257 residual=1.1244163028865358e-06
   DEBUG:root:i=258 residual=1.1244050880647354e-06
   DEBUG:root:i=259 residual=1.1243938609256524e-06
   DEBUG:root:i=260 residual=1.1243826337823709e-06
   DEBUG:root:i=261 residual=1.124371425093287e-06
   DEBUG:root:i=262 residual=1.1243602348703136e-06
   DEBUG:root:i=263 residual=1.124349013896258e-06
   DEBUG:root:i=264 residual=1.1243378052182268e-06
   DEBUG:root:i=265 residual=1.1243266088388188e-06
   DEBUG:root:i=266 residual=1.1243154063199071e-06
   DEBUG:root:i=267 residual=1.1243041853381915e-06
   DEBUG:root:i=268 residual=1.1242929335932912e-06
   DEBUG:root:i=269 residual=1.1242817495036235e-06
   DEBUG:root:i=270 residual=1.1242705346810937e-06
   DEBUG:root:i=271 residual=1.1242593383063903e-06
   DEBUG:root:i=272 residual=1.1242481111758365e-06
   DEBUG:root:i=273 residual=1.124236902489936e-06
   DEBUG:root:i=274 residual=1.1242257184155398e-06
   DEBUG:root:i=275 residual=1.12421446669392e-06
   DEBUG:root:i=276 residual=1.1242032764427047e-06
   DEBUG:root:i=277 residual=1.1241920739289907e-06
   DEBUG:root:i=278 residual=1.124180846795644e-06
   DEBUG:root:i=279 residual=1.1241696688681847e-06
   DEBUG:root:i=280 residual=1.1241584540567957e-06
   DEBUG:root:i=281 residual=1.1241472822812191e-06
   DEBUG:root:i=282 residual=1.1241360613233728e-06
   DEBUG:root:i=283 residual=1.1241248095780772e-06
   DEBUG:root:i=284 residual=1.124113607023556e-06
   DEBUG:root:i=285 residual=1.1241024598696056e-06
   DEBUG:root:i=286 residual=1.1240912266132297e-06
   DEBUG:root:i=287 residual=1.124080011773636e-06
   DEBUG:root:i=288 residual=1.1240687784831616e-06
   DEBUG:root:i=289 residual=1.1240576067050672e-06
   DEBUG:root:i=290 residual=1.124046404196841e-06
   DEBUG:root:i=291 residual=1.1240352016728765e-06
   DEBUG:root:i=292 residual=1.1240240053067383e-06
   DEBUG:root:i=293 residual=1.1240128027801754e-06
   DEBUG:root:i=294 residual=1.1240015941101046e-06
   DEBUG:root:i=295 residual=1.123990373126088e-06
   DEBUG:root:i=296 residual=1.1239791705946031e-06
   DEBUG:root:i=297 residual=1.123967968070864e-06
   DEBUG:root:i=298 residual=1.1239567470887304e-06
   DEBUG:root:i=299 residual=1.1239455753214477e-06
   DEBUG:root:i=300 residual=1.1239343666564023e-06
   DEBUG:root:i=301 residual=1.1239231579807894e-06
   DEBUG:root:i=302 residual=1.1239119493058973e-06
   DEBUG:root:i=303 residual=1.1239007652318599e-06
   DEBUG:root:i=304 residual=1.123889531960066e-06
   DEBUG:root:i=305 residual=1.123878323274215e-06
   DEBUG:root:i=306 residual=1.1238671391979375e-06
   DEBUG:root:i=307 residual=1.1238559305299023e-06
   DEBUG:root:i=308 residual=1.1238447341608242e-06
   DEBUG:root:i=309 residual=1.1238335316372942e-06
   DEBUG:root:i=310 residual=1.1238223352676755e-06
   DEBUG:root:i=311 residual=1.1238111265949414e-06
   DEBUG:root:i=312 residual=1.1237999240741676e-06
   DEBUG:root:i=313 residual=1.1237887153958998e-06
   DEBUG:root:i=314 residual=1.123777512867041e-06
   DEBUG:root:i=315 residual=1.1237663164952629e-06
   DEBUG:root:i=316 residual=1.123755089374673e-06
   DEBUG:root:i=317 residual=1.1237438806813873e-06
   DEBUG:root:i=318 residual=1.1237326720054033e-06
   DEBUG:root:i=319 residual=1.1237214817828336e-06
   DEBUG:root:i=320 residual=1.1237102423535946e-06
   DEBUG:root:i=321 residual=1.1236990644189555e-06
   DEBUG:root:i=322 residual=1.123687849606748e-06
   DEBUG:root:i=323 residual=1.1236766409239197e-06
   DEBUG:root:i=324 residual=1.1236654568527189e-06
   DEBUG:root:i=325 residual=1.1236542604907993e-06
   DEBUG:root:i=326 residual=1.1236430518184392e-06
   DEBUG:root:i=327 residual=1.1236318369914792e-06
   DEBUG:root:i=328 residual=1.1236206467659857e-06
   DEBUG:root:i=329 residual=1.1236094442447107e-06
   DEBUG:root:i=330 residual=1.1235982663333578e-06
   DEBUG:root:i=331 residual=1.1235870269095875e-06
   DEBUG:root:i=332 residual=1.1235758243707718e-06
   DEBUG:root:i=333 residual=1.1235646402990706e-06
   DEBUG:root:i=334 residual=1.123553413181659e-06
   DEBUG:root:i=335 residual=1.1235422414037234e-06
   DEBUG:root:i=336 residual=1.1235310450471257e-06
   DEBUG:root:i=337 residual=1.1235198486808378e-06
   DEBUG:root:i=338 residual=1.1235086215577697e-06
   DEBUG:root:i=339 residual=1.123497419016468e-06
   DEBUG:root:i=340 residual=1.1234862287984075e-06
   DEBUG:root:i=341 residual=1.1234750078272829e-06
   DEBUG:root:i=342 residual=1.1234638052959423e-06
   DEBUG:root:i=343 residual=1.1234526150704995e-06
   DEBUG:root:i=344 residual=1.1234414187112421e-06
   DEBUG:root:i=345 residual=1.1234302038869512e-06
   DEBUG:root:i=346 residual=1.1234190444204082e-06
   DEBUG:root:i=347 residual=1.1234078111586077e-06
   DEBUG:root:i=348 residual=1.123396608622569e-06
   DEBUG:root:i=349 residual=1.1233853876437107e-06
   DEBUG:root:i=350 residual=1.123374203564146e-06
   DEBUG:root:i=351 residual=1.123363050264901e-06
   DEBUG:root:i=352 residual=1.1233518170113013e-06
   DEBUG:root:i=353 residual=1.1233406206267261e-06
   DEBUG:root:i=354 residual=1.123329411951746e-06
   DEBUG:root:i=355 residual=1.1233182032731266e-06
   DEBUG:root:i=356 residual=1.1233070130529904e-06
   DEBUG:root:i=357 residual=1.123295822837985e-06
   DEBUG:root:i=358 residual=1.1232846080186186e-06
   DEBUG:root:i=359 residual=1.1232734300969597e-06
   DEBUG:root:i=360 residual=1.1232622214370819e-06
   DEBUG:root:i=361 residual=1.1232509881500066e-06
   DEBUG:root:i=362 residual=1.1232398040656837e-06
   DEBUG:root:i=363 residual=1.1232286446144132e-06
   DEBUG:root:i=364 residual=1.1232174359624843e-06
   DEBUG:root:i=365 residual=1.1232062395883446e-06
   DEBUG:root:i=366 residual=1.1231949878535185e-06
   DEBUG:root:i=367 residual=1.123183834523042e-06
   DEBUG:root:i=368 residual=1.1231726320221981e-06
   DEBUG:root:i=369 residual=1.1231614664120613e-06
   DEBUG:root:i=370 residual=1.1231502269963297e-06
   DEBUG:root:i=371 residual=1.1231390244599e-06
   DEBUG:root:i=372 residual=1.1231278526895957e-06
   DEBUG:root:i=373 residual=1.12311661942548e-06
   DEBUG:root:i=374 residual=1.123105484557819e-06
   DEBUG:root:i=375 residual=1.1230942205482845e-06
   DEBUG:root:i=376 residual=1.1230830610610342e-06
   DEBUG:root:i=377 residual=1.1230718708614439e-06
   DEBUG:root:i=378 residual=1.1230606683456588e-06
   DEBUG:root:i=379 residual=1.12304945967318e-06
   DEBUG:root:i=380 residual=1.1230382386893704e-06
   DEBUG:root:i=381 residual=1.123027103826357e-06
   DEBUG:root:i=382 residual=1.1230158828817267e-06
   DEBUG:root:i=383 residual=1.1230047172635797e-06
   DEBUG:root:i=384 residual=1.1229934655470444e-06
   DEBUG:root:i=385 residual=1.1229822876047854e-06
   DEBUG:root:i=386 residual=1.1229711035462579e-06
   DEBUG:root:i=387 residual=1.1229598641280519e-06
   DEBUG:root:i=388 residual=1.122948698494243e-06
   DEBUG:root:i=389 residual=1.122937526746875e-06
   DEBUG:root:i=390 residual=1.1229263365475908e-06
   DEBUG:root:i=391 residual=1.1229151340301583e-06
   DEBUG:root:i=392 residual=1.1229039438099222e-06
   DEBUG:root:i=393 residual=1.1228927289934919e-06
   DEBUG:root:i=394 residual=1.1228815080095064e-06
   DEBUG:root:i=395 residual=1.1228703300794665e-06
   DEBUG:root:i=396 residual=1.122859127568723e-06
   DEBUG:root:i=397 residual=1.1228479373538688e-06
   DEBUG:root:i=398 residual=1.1228367409872075e-06
   DEBUG:root:i=399 residual=1.122825538466084e-06
   DEBUG:root:i=400 residual=1.1228143544008476e-06
   DEBUG:root:i=401 residual=1.1228031518873373e-06
   DEBUG:root:i=402 residual=1.122791961669968e-06
   DEBUG:root:i=403 residual=1.1227807776068166e-06
   DEBUG:root:i=404 residual=1.1227695750963078e-06
   DEBUG:root:i=405 residual=1.1227583479634768e-06
   DEBUG:root:i=406 residual=1.1227471638820386e-06
   DEBUG:root:i=407 residual=1.1227359921268629e-06
   DEBUG:root:i=408 residual=1.122724789624345e-06
   DEBUG:root:i=409 residual=1.122713599404452e-06
   DEBUG:root:i=410 residual=1.122702403040706e-06
   DEBUG:root:i=411 residual=1.1226911943681592e-06
   DEBUG:root:i=412 residual=1.1226799979933969e-06
   DEBUG:root:i=413 residual=1.122668820082248e-06
   DEBUG:root:i=414 residual=1.1226576175739e-06
   DEBUG:root:i=415 residual=1.122646433508208e-06
   DEBUG:root:i=416 residual=1.1226352802040315e-06
   DEBUG:root:i=417 residual=1.122624065402765e-06
   DEBUG:root:i=418 residual=1.122612887487376e-06
   DEBUG:root:i=419 residual=1.122601660364174e-06
   DEBUG:root:i=420 residual=1.122590494743531e-06
   DEBUG:root:i=421 residual=1.12257931684727e-06
   DEBUG:root:i=422 residual=1.1225681143371293e-06
   DEBUG:root:i=423 residual=1.122556911816666e-06
   DEBUG:root:i=424 residual=1.1225457154398782e-06
   DEBUG:root:i=425 residual=1.1225345498390949e-06
   DEBUG:root:i=426 residual=1.1225233411797308e-06
   DEBUG:root:i=427 residual=1.1225121263499104e-06
   DEBUG:root:i=428 residual=1.1225009545775447e-06
   DEBUG:root:i=429 residual=1.1224897766813084e-06
   DEBUG:root:i=430 residual=1.1224785803228238e-06
   DEBUG:root:i=431 residual=1.1224673962601295e-06
   DEBUG:root:i=432 residual=1.1224561937470668e-06
   DEBUG:root:i=433 residual=1.1224450035240355e-06
   DEBUG:root:i=434 residual=1.122433801013998e-06
   DEBUG:root:i=435 residual=1.12242262924646e-06
   DEBUG:root:i=436 residual=1.1224114390443508e-06
   DEBUG:root:i=437 residual=1.1224002611361146e-06
   DEBUG:root:i=438 residual=1.1223890586278443e-06
   DEBUG:root:i=439 residual=1.1223778499480897e-06
   DEBUG:root:i=440 residual=1.1223666474294858e-06
   DEBUG:root:i=441 residual=1.1223554756595015e-06
   DEBUG:root:i=442 residual=1.122344328522012e-06
   DEBUG:root:i=443 residual=1.1223331383306111e-06
   DEBUG:root:i=444 residual=1.1223219358157866e-06
   DEBUG:root:i=445 residual=1.1223107209916233e-06
   DEBUG:root:i=446 residual=1.1222995307635731e-06
   DEBUG:root:i=447 residual=1.1222883713125826e-06
   DEBUG:root:i=448 residual=1.1222771503544883e-06
   DEBUG:root:i=449 residual=1.1222659724330279e-06
   DEBUG:root:i=450 residual=1.1222547760692336e-06
   DEBUG:root:i=451 residual=1.1222435797049136e-06
   DEBUG:root:i=452 residual=1.1222324202511835e-06
   DEBUG:root:i=453 residual=1.1222212362016571e-06
   DEBUG:root:i=454 residual=1.122210045994666e-06
   DEBUG:root:i=455 residual=1.1221988557828505e-06
   DEBUG:root:i=456 residual=1.122187647115474e-06
   DEBUG:root:i=457 residual=1.1221764261339952e-06
   DEBUG:root:i=458 residual=1.1221652789685223e-06
   DEBUG:root:i=459 residual=1.1221540703187602e-06
   DEBUG:root:i=460 residual=1.122142886253292e-06
   DEBUG:root:i=461 residual=1.122131708344562e-06
   DEBUG:root:i=462 residual=1.122120475075607e-06
   DEBUG:root:i=463 residual=1.1221092971433148e-06
   DEBUG:root:i=464 residual=1.122098131545538e-06
   DEBUG:root:i=465 residual=1.1220869351949848e-06
   DEBUG:root:i=466 residual=1.122075751129281e-06
   DEBUG:root:i=467 residual=1.1220645732284022e-06
   DEBUG:root:i=468 residual=1.1220534014742314e-06
   DEBUG:root:i=469 residual=1.1220422051209253e-06
   DEBUG:root:i=470 residual=1.1220310149061777e-06
   DEBUG:root:i=471 residual=1.122019855450122e-06
   DEBUG:root:i=472 residual=1.1220086160425388e-06
   DEBUG:root:i=473 residual=1.1219974258018703e-06
   DEBUG:root:i=474 residual=1.1219862355893009e-06
   DEBUG:root:i=475 residual=1.12197503307081e-06
   DEBUG:root:i=476 residual=1.1219638982243258e-06
   DEBUG:root:i=477 residual=1.1219527080354468e-06
   DEBUG:root:i=478 residual=1.1219415485823988e-06
   DEBUG:root:i=479 residual=1.1219303276300068e-06
   DEBUG:root:i=480 residual=1.1219191558604874e-06
   DEBUG:root:i=481 residual=1.1219080087127408e-06
   DEBUG:root:i=482 residual=1.1218967877680934e-06
   DEBUG:root:i=483 residual=1.1218855790880426e-06
   DEBUG:root:i=484 residual=1.1218744134672645e-06
   DEBUG:root:i=485 residual=1.121863229417117e-06
   DEBUG:root:i=486 residual=1.1218520638220007e-06
   DEBUG:root:i=487 residual=1.121840879769561e-06
   DEBUG:root:i=488 residual=1.1218296957149725e-06
   DEBUG:root:i=489 residual=1.1218185116596406e-06
   DEBUG:root:i=490 residual=1.1218072906866381e-06
   DEBUG:root:i=491 residual=1.121796137371555e-06
   DEBUG:root:i=492 residual=1.1217849225703324e-06
   DEBUG:root:i=493 residual=1.121773750803686e-06
   DEBUG:root:i=494 residual=1.1217625913550696e-06
   DEBUG:root:i=495 residual=1.1217514196185592e-06
   DEBUG:root:i=496 residual=1.121740217106566e-06
   DEBUG:root:i=497 residual=1.1217290515013808e-06
   DEBUG:root:i=498 residual=1.1217178674538427e-06
   DEBUG:root:i=499 residual=1.1217066772395136e-06
   DEBUG:root:i=500 residual=1.1216954931843523e-06
   DEBUG:root:i=501 residual=1.1216843152783303e-06
   DEBUG:root:i=502 residual=1.1216731066135327e-06
   DEBUG:root:i=503 residual=1.1216619594584534e-06
   DEBUG:root:i=504 residual=1.1216507754187422e-06
   DEBUG:root:i=505 residual=1.1216395790555223e-06
   DEBUG:root:i=506 residual=1.1216283765395553e-06
   DEBUG:root:i=507 residual=1.1216172170785253e-06
   DEBUG:root:i=508 residual=1.1216060268846059e-06
   DEBUG:root:i=509 residual=1.1215948428161109e-06
   DEBUG:root:i=510 residual=1.121583677221824e-06
   DEBUG:root:i=511 residual=1.1215724931745924e-06
   DEBUG:root:i=512 residual=1.1215613029625509e-06
   DEBUG:root:i=513 residual=1.1215501066014434e-06
   DEBUG:root:i=514 residual=1.1215389286872168e-06
   DEBUG:root:i=515 residual=1.1215277323316193e-06
   DEBUG:root:i=516 residual=1.1215165482658202e-06
   DEBUG:root:i=517 residual=1.1215053765113462e-06
   DEBUG:root:i=518 residual=1.1214941986105645e-06
   DEBUG:root:i=519 residual=1.1214830084063784e-06
   DEBUG:root:i=520 residual=1.1214718551049894e-06
   DEBUG:root:i=521 residual=1.1214606587615745e-06
   DEBUG:root:i=522 residual=1.1214494562469235e-06
   DEBUG:root:i=523 residual=1.12143830293463e-06
   DEBUG:root:i=524 residual=1.1214271065886242e-06
   DEBUG:root:i=525 residual=1.1214158794639694e-06
   DEBUG:root:i=526 residual=1.1214047445963667e-06
   DEBUG:root:i=527 residual=1.1213935851711761e-06
   DEBUG:root:i=528 residual=1.1213823765226576e-06
   DEBUG:root:i=529 residual=1.1213712047559245e-06
   DEBUG:root:i=530 residual=1.121360026854705e-06
   DEBUG:root:i=531 residual=1.1213488182006691e-06
   DEBUG:root:i=532 residual=1.121337664883752e-06
   DEBUG:root:i=533 residual=1.1213264685375007e-06
   DEBUG:root:i=534 residual=1.1213152783257779e-06
   DEBUG:root:i=535 residual=1.1213041434794042e-06
   DEBUG:root:i=536 residual=1.1212929471413572e-06
   DEBUG:root:i=537 residual=1.1212817569300378e-06
   DEBUG:root:i=538 residual=1.1212705851700868e-06
   DEBUG:root:i=539 residual=1.1212594257275239e-06
   DEBUG:root:i=540 residual=1.121248223224755e-06
   DEBUG:root:i=541 residual=1.1212370576200506e-06
   DEBUG:root:i=542 residual=1.1212258797220184e-06
   DEBUG:root:i=543 residual=1.1212147079676503e-06
   DEBUG:root:i=544 residual=1.121203517771517e-06
   DEBUG:root:i=545 residual=1.1211923337087527e-06
   DEBUG:root:i=546 residual=1.121181137347468e-06
   DEBUG:root:i=547 residual=1.1211699778887292e-06
   DEBUG:root:i=548 residual=1.121158830759416e-06
   DEBUG:root:i=549 residual=1.1211476405662421e-06
   DEBUG:root:i=550 residual=1.1211364319042937e-06
   DEBUG:root:i=551 residual=1.1211252478313365e-06
   DEBUG:root:i=552 residual=1.1211140760763759e-06
   DEBUG:root:i=553 residual=1.1211028981784684e-06
   DEBUG:root:i=554 residual=1.121091720275137e-06
   DEBUG:root:i=555 residual=1.12108056082917e-06
   DEBUG:root:i=556 residual=1.1210693706334348e-06
   DEBUG:root:i=557 residual=1.1210581804186938e-06
   DEBUG:root:i=558 residual=1.1210470517292395e-06
   DEBUG:root:i=559 residual=1.1210358492424482e-06
   DEBUG:root:i=560 residual=1.1210246528738369e-06
   DEBUG:root:i=561 residual=1.1210134995698676e-06
   DEBUG:root:i=562 residual=1.1210023401339705e-06
   DEBUG:root:i=563 residual=1.120991149940735e-06
   DEBUG:root:i=564 residual=1.1209799720302574e-06
   DEBUG:root:i=565 residual=1.120968800278233e-06
   DEBUG:root:i=566 residual=1.1209576100794364e-06
   DEBUG:root:i=567 residual=1.1209464383226614e-06
   DEBUG:root:i=568 residual=1.1209352542726912e-06
   DEBUG:root:i=569 residual=1.1209240886704014e-06
   DEBUG:root:i=570 residual=1.1209129046204844e-06
   DEBUG:root:i=571 residual=1.1209017513242642e-06
   DEBUG:root:i=572 residual=1.1208905918891123e-06
   DEBUG:root:i=573 residual=1.1208793709343136e-06
   DEBUG:root:i=574 residual=1.1208681991647666e-06
   DEBUG:root:i=575 residual=1.1208570274156255e-06
   DEBUG:root:i=576 residual=1.1208458433630124e-06
   DEBUG:root:i=577 residual=1.120834665464869e-06
   DEBUG:root:i=578 residual=1.1208235060113505e-06
   DEBUG:root:i=579 residual=1.120812334273163e-06
   DEBUG:root:i=580 residual=1.1208011502207644e-06
   DEBUG:root:i=581 residual=1.1207899661633666e-06
   DEBUG:root:i=582 residual=1.1207788313222138e-06
   DEBUG:root:i=583 residual=1.1207676411387308e-06
   DEBUG:root:i=584 residual=1.1207564939894206e-06
   DEBUG:root:i=585 residual=1.1207452607387974e-06
   DEBUG:root:i=586 residual=1.1207341320260343e-06
   DEBUG:root:i=587 residual=1.1207229295389446e-06
   DEBUG:root:i=588 residual=1.1207117454684421e-06
   DEBUG:root:i=589 residual=1.1207005798686658e-06
   DEBUG:root:i=590 residual=1.1206894204332674e-06
   DEBUG:root:i=591 residual=1.1206782671423618e-06
   DEBUG:root:i=592 residual=1.1206670769515817e-06
   DEBUG:root:i=593 residual=1.1206558805933887e-06
   DEBUG:root:i=594 residual=1.1206447272842296e-06
   DEBUG:root:i=595 residual=1.1206335432389278e-06
   DEBUG:root:i=596 residual=1.1206223899482673e-06
   DEBUG:root:i=597 residual=1.120611242813849e-06
   DEBUG:root:i=598 residual=1.120600034170608e-06
   DEBUG:root:i=599 residual=1.1205888808613472e-06
   DEBUG:root:i=600 residual=1.120577727575421e-06
   DEBUG:root:i=601 residual=1.1205665066313607e-06
   DEBUG:root:i=602 residual=1.1205553410054187e-06
   DEBUG:root:i=603 residual=1.1205442000224935e-06
   DEBUG:root:i=604 residual=1.1205330282869911e-06
   DEBUG:root:i=605 residual=1.1205218380905643e-06
   DEBUG:root:i=606 residual=1.120510660179849e-06
   DEBUG:root:i=607 residual=1.1204994884311596e-06
   DEBUG:root:i=608 residual=1.1204883658980857e-06
   DEBUG:root:i=609 residual=1.1204771634144955e-06
   DEBUG:root:i=610 residual=1.1204660101132924e-06
   DEBUG:root:i=611 residual=1.1204548322226851e-06
   DEBUG:root:i=612 residual=1.1204436481658588e-06
   DEBUG:root:i=613 residual=1.1204324641128884e-06
   DEBUG:root:i=614 residual=1.1204213354211159e-06
   DEBUG:root:i=615 residual=1.1204101329320242e-06
   DEBUG:root:i=616 residual=1.1203990103916698e-06
   DEBUG:root:i=617 residual=1.1203878079075874e-06
   DEBUG:root:i=618 residual=1.1203766299972288e-06
   DEBUG:root:i=619 residual=1.1203654520912936e-06
   DEBUG:root:i=620 residual=1.1203543049492899e-06
   DEBUG:root:i=621 residual=1.120343096305248e-06
   DEBUG:root:i=622 residual=1.1203319491454979e-06
   DEBUG:root:i=623 residual=1.1203207774118397e-06
   DEBUG:root:i=624 residual=1.1203095933596043e-06
   DEBUG:root:i=625 residual=1.12029844006615e-06
   DEBUG:root:i=626 residual=1.1202872867773606e-06
   DEBUG:root:i=627 residual=1.1202761211961277e-06
   DEBUG:root:i=628 residual=1.1202649309976673e-06
   DEBUG:root:i=629 residual=1.1202537653928066e-06
   DEBUG:root:i=630 residual=1.1202426059552973e-06
   DEBUG:root:i=631 residual=1.1202314219081155e-06
   DEBUG:root:i=632 residual=1.1202202317014452e-06
   DEBUG:root:i=633 residual=1.1202091214619644e-06
   DEBUG:root:i=634 residual=1.1201979435951623e-06
   DEBUG:root:i=635 residual=1.1201867472400773e-06
   DEBUG:root:i=636 residual=1.1201755816324326e-06
   DEBUG:root:i=637 residual=1.12016441604041e-06
   DEBUG:root:i=638 residual=1.1201532750527446e-06
   DEBUG:root:i=639 residual=1.1201421094740456e-06
   DEBUG:root:i=640 residual=1.1201309131265146e-06
   DEBUG:root:i=641 residual=1.1201197413695503e-06
   DEBUG:root:i=642 residual=1.1201085634668603e-06
   DEBUG:root:i=643 residual=1.1200974163219314e-06
   DEBUG:root:i=644 residual=1.1200862753470063e-06
   DEBUG:root:i=645 residual=1.1200751036221951e-06
   DEBUG:root:i=646 residual=1.1200639195701264e-06
   DEBUG:root:i=647 residual=1.12005278472421e-06
   DEBUG:root:i=648 residual=1.1200415822400855e-06
   DEBUG:root:i=649 residual=1.1200304350876159e-06
   DEBUG:root:i=650 residual=1.1200192448943743e-06
   DEBUG:root:i=651 residual=1.1200080977495637e-06
   DEBUG:root:i=652 residual=1.1199969506209057e-06
   DEBUG:root:i=653 residual=1.1199857850340572e-06
   DEBUG:root:i=654 residual=1.1199746256022949e-06
   DEBUG:root:i=655 residual=1.1199634415528357e-06
   DEBUG:root:i=656 residual=1.1199522944134707e-06
   DEBUG:root:i=657 residual=1.1199411042175959e-06
   DEBUG:root:i=658 residual=1.119929969376067e-06
   DEBUG:root:i=659 residual=1.119918766887039e-06
   DEBUG:root:i=660 residual=1.1199076320388856e-06
   DEBUG:root:i=661 residual=1.1198964356981688e-06
   DEBUG:root:i=662 residual=1.1198853316162007e-06
   DEBUG:root:i=663 residual=1.1198741168361234e-06
   DEBUG:root:i=664 residual=1.1198629635249701e-06
   DEBUG:root:i=665 residual=1.119851810244298e-06
   DEBUG:root:i=666 residual=1.1198406323542875e-06
   DEBUG:root:i=667 residual=1.1198294544534868e-06
   DEBUG:root:i=668 residual=1.11981827040166e-06
   DEBUG:root:i=669 residual=1.1198070863384978e-06
   DEBUG:root:i=670 residual=1.119795963806141e-06
   DEBUG:root:i=671 residual=1.1197847920804067e-06
   DEBUG:root:i=672 residual=1.1197736080341696e-06
   DEBUG:root:i=673 residual=1.1197624608843524e-06
   DEBUG:root:i=674 residual=1.1197512891536722e-06
   DEBUG:root:i=675 residual=1.1197401112532726e-06
   DEBUG:root:i=676 residual=1.1197289518104739e-06
   DEBUG:root:i=677 residual=1.1197178354325622e-06
   DEBUG:root:i=678 residual=1.1197066575659675e-06
   DEBUG:root:i=679 residual=1.1196954858149377e-06
   DEBUG:root:i=680 residual=1.1196843263727472e-06
   DEBUG:root:i=681 residual=1.1196731361763286e-06
   DEBUG:root:i=682 residual=1.1196619890292522e-06
   DEBUG:root:i=683 residual=1.119650780382981e-06
   DEBUG:root:i=684 residual=1.1196396578325108e-06
   DEBUG:root:i=685 residual=1.1196284984127739e-06
   DEBUG:root:i=686 residual=1.1196173451274647e-06
   DEBUG:root:i=687 residual=1.1196061856980978e-06
   DEBUG:root:i=688 residual=1.1195950324123734e-06
   DEBUG:root:i=689 residual=1.1195838914328427e-06
   DEBUG:root:i=690 residual=1.1195727074018727e-06
   DEBUG:root:i=691 residual=1.1195615664095218e-06
   DEBUG:root:i=692 residual=1.1195503823728102e-06
   DEBUG:root:i=693 residual=1.119539229076591e-06
   DEBUG:root:i=694 residual=1.1195280573358808e-06
   DEBUG:root:i=695 residual=1.119516885594791e-06
   DEBUG:root:i=696 residual=1.1195057384505289e-06
   DEBUG:root:i=697 residual=1.1194945728740205e-06
   DEBUG:root:i=698 residual=1.1194834257326642e-06
   DEBUG:root:i=699 residual=1.1194722724571077e-06
   DEBUG:root:i=700 residual=1.119461106871132e-06
   DEBUG:root:i=701 residual=1.1194499351298527e-06
   DEBUG:root:i=702 residual=1.1194387756875707e-06
   DEBUG:root:i=703 residual=1.119427647006412e-06
   DEBUG:root:i=704 residual=1.1194164383756785e-06
   DEBUG:root:i=705 residual=1.119405278910087e-06
   DEBUG:root:i=706 residual=1.1193941256263036e-06
   DEBUG:root:i=707 residual=1.119382978497641e-06
   DEBUG:root:i=708 residual=1.1193717821555584e-06
   DEBUG:root:i=709 residual=1.119360671918545e-06
   DEBUG:root:i=710 residual=1.1193495001959458e-06
   DEBUG:root:i=711 residual=1.119338389970089e-06
   DEBUG:root:i=712 residual=1.1193272121086452e-06
   DEBUG:root:i=713 residual=1.1193160219031555e-06
   DEBUG:root:i=714 residual=1.119304893211141e-06
   DEBUG:root:i=715 residual=1.1192936722691383e-06
   DEBUG:root:i=716 residual=1.1192825681689607e-06
   DEBUG:root:i=717 residual=1.1192713718491171e-06
   DEBUG:root:i=718 residual=1.119260230849395e-06
   DEBUG:root:i=719 residual=1.1192490775739473e-06
   DEBUG:root:i=720 residual=1.1192378873803766e-06
   DEBUG:root:i=721 residual=1.1192267525397521e-06
   DEBUG:root:i=722 residual=1.1192155623538168e-06
   DEBUG:root:i=723 residual=1.1192044275076665e-06
   DEBUG:root:i=724 residual=1.1191932803868213e-06
   DEBUG:root:i=725 residual=1.1191821209623884e-06
   DEBUG:root:i=726 residual=1.1191709984311178e-06
   DEBUG:root:i=727 residual=1.1191598144096731e-06
   DEBUG:root:i=728 residual=1.1191486795693147e-06
   DEBUG:root:i=729 residual=1.1191374893863952e-06
   DEBUG:root:i=730 residual=1.119126317629976e-06
   DEBUG:root:i=731 residual=1.1191151643387852e-06
   DEBUG:root:i=732 residual=1.1191040172048844e-06
   DEBUG:root:i=733 residual=1.119092839324912e-06
   DEBUG:root:i=734 residual=1.1190816921838856e-06
   DEBUG:root:i=735 residual=1.1190705450524108e-06
   DEBUG:root:i=736 residual=1.119059373321783e-06
   DEBUG:root:i=737 residual=1.1190482384867526e-06
   DEBUG:root:i=738 residual=1.1190370913653363e-06
   DEBUG:root:i=739 residual=1.1190259442395908e-06
   DEBUG:root:i=740 residual=1.1190147971136542e-06
   DEBUG:root:i=741 residual=1.1190036438360615e-06
   DEBUG:root:i=742 residual=1.118992465946536e-06
   DEBUG:root:i=743 residual=1.118981294200462e-06
   DEBUG:root:i=744 residual=1.1189701409069594e-06
   DEBUG:root:i=745 residual=1.1189589753228178e-06
   DEBUG:root:i=746 residual=1.1189478712485895e-06
   DEBUG:root:i=747 residual=1.1189366872326253e-06
   DEBUG:root:i=748 residual=1.1189255154821222e-06
   DEBUG:root:i=749 residual=1.1189143560418961e-06
   DEBUG:root:i=750 residual=1.118903215059953e-06
   DEBUG:root:i=751 residual=1.118892049478653e-06
   DEBUG:root:i=752 residual=1.1188808900383868e-06
   DEBUG:root:i=753 residual=1.1188697490590027e-06
   DEBUG:root:i=754 residual=1.1188586080900915e-06
   DEBUG:root:i=755 residual=1.1188474302050524e-06
   DEBUG:root:i=756 residual=1.118836270760595e-06
   DEBUG:root:i=757 residual=1.118825117474405e-06
   DEBUG:root:i=758 residual=1.1188139949556238e-06
   DEBUG:root:i=759 residual=1.118802829384689e-06
   DEBUG:root:i=760 residual=1.1187916760995402e-06
   DEBUG:root:i=761 residual=1.1187804920578368e-06
   DEBUG:root:i=762 residual=1.1187693572146793e-06
   DEBUG:root:i=763 residual=1.118758203944241e-06
   DEBUG:root:i=764 residual=1.1187470506640783e-06
   DEBUG:root:i=765 residual=1.1187359158363583e-06
   DEBUG:root:i=766 residual=1.1187247502604512e-06
   DEBUG:root:i=767 residual=1.1187135846716204e-06
   DEBUG:root:i=768 residual=1.118702425234019e-06
   DEBUG:root:i=769 residual=1.1186912596426766e-06
   DEBUG:root:i=770 residual=1.1186801555709198e-06
   DEBUG:root:i=771 residual=1.1186689777042452e-06
   DEBUG:root:i=772 residual=1.1186578059611494e-06
   DEBUG:root:i=773 residual=1.1186466649714122e-06
   DEBUG:root:i=774 residual=1.1186355239967005e-06
   DEBUG:root:i=775 residual=1.1186243461201895e-06
   DEBUG:root:i=776 residual=1.11861321743391e-06
   DEBUG:root:i=777 residual=1.1186020641584522e-06
   DEBUG:root:i=778 residual=1.118590880124945e-06
   DEBUG:root:i=779 residual=1.1185797391297585e-06
   DEBUG:root:i=780 residual=1.1185685735484077e-06
   DEBUG:root:i=781 residual=1.1185574079620837e-06
   DEBUG:root:i=782 residual=1.1185462792779833e-06
   DEBUG:root:i=783 residual=1.1185351444658624e-06
   DEBUG:root:i=784 residual=1.1185239665866202e-06
   DEBUG:root:i=785 residual=1.1185128194453038e-06
   DEBUG:root:i=786 residual=1.1185016969261858e-06
   DEBUG:root:i=787 residual=1.1184905436636735e-06
   DEBUG:root:i=788 residual=1.118479402682279e-06
   DEBUG:root:i=789 residual=1.1184682124996438e-06
   DEBUG:root:i=790 residual=1.1184570592011751e-06
   DEBUG:root:i=791 residual=1.1184459489748021e-06
   DEBUG:root:i=792 residual=1.118434771115978e-06
   DEBUG:root:i=793 residual=1.1184236301238507e-06
   DEBUG:root:i=794 residual=1.1184125076073683e-06
   DEBUG:root:i=795 residual=1.1184013358875855e-06
   DEBUG:root:i=796 residual=1.1183902072069732e-06
   DEBUG:root:i=797 residual=1.1183790539339331e-06
   DEBUG:root:i=798 residual=1.1183678822018468e-06
   DEBUG:root:i=799 residual=1.1183566920004174e-06
   DEBUG:root:i=800 residual=1.1183455694604113e-06
   DEBUG:root:i=801 residual=1.1183344346453848e-06
   DEBUG:root:i=802 residual=1.1183232936791972e-06
   DEBUG:root:i=803 residual=1.1183121096480814e-06
   DEBUG:root:i=804 residual=1.1183009563497626e-06
   DEBUG:root:i=805 residual=1.1182897969174601e-06
   DEBUG:root:i=806 residual=1.1182786743879698e-06
   DEBUG:root:i=807 residual=1.1182675149765384e-06
   DEBUG:root:i=808 residual=1.1182563678404091e-06
   DEBUG:root:i=809 residual=1.1182452453217656e-06
   DEBUG:root:i=810 residual=1.1182340920541438e-06
   DEBUG:root:i=811 residual=1.1182229141751233e-06
   DEBUG:root:i=812 residual=1.1182117793297893e-06
   DEBUG:root:i=813 residual=1.1182006199072282e-06
   DEBUG:root:i=814 residual=1.118189497386009e-06
   DEBUG:root:i=815 residual=1.1181783441161015e-06
   DEBUG:root:i=816 residual=1.1181671785328412e-06
   DEBUG:root:i=817 residual=1.1181560314014403e-06
   DEBUG:root:i=818 residual=1.1181448842703813e-06
   DEBUG:root:i=819 residual=1.1181337494504854e-06
   DEBUG:root:i=820 residual=1.1181226146332518e-06
   DEBUG:root:i=821 residual=1.118111442908376e-06
   DEBUG:root:i=822 residual=1.1181002773143288e-06
   DEBUG:root:i=823 residual=1.1180891117201855e-06
   DEBUG:root:i=824 residual=1.118077976889748e-06
   DEBUG:root:i=825 residual=1.1180668174649922e-06
   DEBUG:root:i=826 residual=1.1180556949410442e-06
   DEBUG:root:i=827 residual=1.118044529373097e-06
   DEBUG:root:i=828 residual=1.1180333945399549e-06
   DEBUG:root:i=829 residual=1.118022278180789e-06
   DEBUG:root:i=830 residual=1.1180111003093815e-06
   DEBUG:root:i=831 residual=1.1179999470214804e-06
   DEBUG:root:i=832 residual=1.1179888244951122e-06
   DEBUG:root:i=833 residual=1.117977671229769e-06
   DEBUG:root:i=834 residual=1.1179665364051736e-06
   DEBUG:root:i=835 residual=1.1179553769838885e-06
   DEBUG:root:i=836 residual=1.1179442298474928e-06
   DEBUG:root:i=837 residual=1.1179331011793734e-06
   DEBUG:root:i=838 residual=1.1179219540665884e-06
   DEBUG:root:i=839 residual=1.1179108315476152e-06
   DEBUG:root:i=840 residual=1.117899653671416e-06
   DEBUG:root:i=841 residual=1.1178885065322869e-06
   DEBUG:root:i=842 residual=1.1178773840105539e-06
   DEBUG:root:i=843 residual=1.1178662122934308e-06
   DEBUG:root:i=844 residual=1.117855102065222e-06
   DEBUG:root:i=845 residual=1.1178439118951536e-06
   DEBUG:root:i=846 residual=1.1178327832040703e-06
   DEBUG:root:i=847 residual=1.1178216237821366e-06
   DEBUG:root:i=848 residual=1.1178104828030294e-06
   DEBUG:root:i=849 residual=1.1177993295279265e-06
   DEBUG:root:i=850 residual=1.1177881947027604e-06
   DEBUG:root:i=851 residual=1.1177770414302107e-06
   DEBUG:root:i=852 residual=1.117765931209955e-06
   DEBUG:root:i=853 residual=1.1177547902565057e-06
   DEBUG:root:i=854 residual=1.1177436431360118e-06
   DEBUG:root:i=855 residual=1.1177325267691488e-06
   DEBUG:root:i=856 residual=1.1177213673555681e-06
   DEBUG:root:i=857 residual=1.1177102079213321e-06
   DEBUG:root:i=858 residual=1.1176990607904324e-06
   DEBUG:root:i=859 residual=1.1176879259622369e-06
   DEBUG:root:i=860 residual=1.1176767788442692e-06
   DEBUG:root:i=861 residual=1.1176656624826752e-06
   DEBUG:root:i=862 residual=1.117654509217988e-06
   DEBUG:root:i=863 residual=1.117643349786259e-06
   DEBUG:root:i=864 residual=1.1176321965008654e-06
   DEBUG:root:i=865 residual=1.1176210985891643e-06
   DEBUG:root:i=866 residual=1.117609969941558e-06
   DEBUG:root:i=867 residual=1.1175988228239185e-06
   DEBUG:root:i=868 residual=1.1175876203356367e-06
   DEBUG:root:i=869 residual=1.117576491638652e-06
   DEBUG:root:i=870 residual=1.1175653506695315e-06
   DEBUG:root:i=871 residual=1.1175542343074347e-06
   DEBUG:root:i=872 residual=1.117543062593172e-06
   DEBUG:root:i=873 residual=1.1175319277524801e-06
   DEBUG:root:i=874 residual=1.1175207990924965e-06
   DEBUG:root:i=875 residual=1.1175096212155698e-06
   DEBUG:root:i=876 residual=1.117498504835301e-06
   DEBUG:root:i=877 residual=1.1174873392671252e-06
   DEBUG:root:i=878 residual=1.117476204440078e-06
   DEBUG:root:i=879 residual=1.1174650819209425e-06
   DEBUG:root:i=880 residual=1.117453959420262e-06
   DEBUG:root:i=881 residual=1.1174428123026796e-06
   DEBUG:root:i=882 residual=1.1174316590329755e-06
   DEBUG:root:i=883 residual=1.1174205303520415e-06
   DEBUG:root:i=884 residual=1.1174094016944158e-06
   DEBUG:root:i=885 residual=1.1173982422783215e-06
   DEBUG:root:i=886 residual=1.1173870766871631e-06
   DEBUG:root:i=887 residual=1.1173759726129234e-06
   DEBUG:root:i=888 residual=1.1173648378135535e-06
   DEBUG:root:i=889 residual=1.1173537029997255e-06
   DEBUG:root:i=890 residual=1.117342543578121e-06
   DEBUG:root:i=891 residual=1.1173313964473914e-06
   DEBUG:root:i=892 residual=1.117320273923332e-06
   DEBUG:root:i=893 residual=1.117309157574136e-06
   DEBUG:root:i=894 residual=1.1172980043099379e-06
   DEBUG:root:i=895 residual=1.1172868448809026e-06
   DEBUG:root:i=896 residual=1.117275746964136e-06
   DEBUG:root:i=897 residual=1.1172645937070826e-06
   DEBUG:root:i=898 residual=1.1172534342778022e-06
   DEBUG:root:i=899 residual=1.117242287147302e-06
   DEBUG:root:i=900 residual=1.1172311830807184e-06
   DEBUG:root:i=901 residual=1.1172200421325026e-06
   DEBUG:root:i=902 residual=1.1172088580969417e-06
   DEBUG:root:i=903 residual=1.1171977478659087e-06
   DEBUG:root:i=904 residual=1.117186613061474e-06
   DEBUG:root:i=905 residual=1.1171754782447132e-06
   DEBUG:root:i=906 residual=1.117164349581923e-06
   DEBUG:root:i=907 residual=1.1171531655560779e-06
   DEBUG:root:i=908 residual=1.117142030713498e-06
   DEBUG:root:i=909 residual=1.1171309081991065e-06
   DEBUG:root:i=910 residual=1.1171197672405456e-06
   DEBUG:root:i=911 residual=1.1171086262697008e-06
   DEBUG:root:i=912 residual=1.117097466843e-06
   DEBUG:root:i=913 residual=1.1170863381667813e-06
   DEBUG:root:i=914 residual=1.1170752095046104e-06
   DEBUG:root:i=915 residual=1.1170640746947366e-06
   DEBUG:root:i=916 residual=1.1170529337261678e-06
   DEBUG:root:i=917 residual=1.1170417619992922e-06
   DEBUG:root:i=918 residual=1.1170306517705691e-06
   DEBUG:root:i=919 residual=1.1170195108119161e-06
   DEBUG:root:i=920 residual=1.1170083575421e-06
   DEBUG:root:i=921 residual=1.116997241170417e-06
   DEBUG:root:i=922 residual=1.1169861125152403e-06
   DEBUG:root:i=923 residual=1.1169749838554504e-06
   DEBUG:root:i=924 residual=1.1169638859567816e-06
   DEBUG:root:i=925 residual=1.1169527142472682e-06
   DEBUG:root:i=926 residual=1.1169415609576072e-06
   DEBUG:root:i=927 residual=1.1169304322891558e-06
   DEBUG:root:i=928 residual=1.1169193097732654e-06
   DEBUG:root:i=929 residual=1.1169081872698378e-06
   DEBUG:root:i=930 residual=1.1168970217044946e-06
   DEBUG:root:i=931 residual=1.116885917628366e-06
   DEBUG:root:i=932 residual=1.1168747766803917e-06
   DEBUG:root:i=933 residual=1.1168636295549332e-06
   DEBUG:root:i=934 residual=1.1168525008869012e-06
   DEBUG:root:i=935 residual=1.1168413660776725e-06
   DEBUG:root:i=936 residual=1.1168302312579555e-06
   DEBUG:root:i=937 residual=1.1168190841402765e-06
   DEBUG:root:i=938 residual=1.1168079800741414e-06
   DEBUG:root:i=939 residual=1.1167968083700716e-06
   DEBUG:root:i=940 residual=1.116785691984944e-06
   DEBUG:root:i=941 residual=1.1167745387281072e-06
   DEBUG:root:i=942 residual=1.116763385448433e-06
   DEBUG:root:i=943 residual=1.1167522875284104e-06
   DEBUG:root:i=944 residual=1.116741152734934e-06
   DEBUG:root:i=945 residual=1.116730017920717e-06
   DEBUG:root:i=946 residual=1.1167188584969742e-06
   DEBUG:root:i=947 residual=1.116707723669382e-06
   DEBUG:root:i=948 residual=1.1166965888547324e-06
   DEBUG:root:i=949 residual=1.1166854724928117e-06
   DEBUG:root:i=950 residual=1.1166743130797523e-06
   DEBUG:root:i=951 residual=1.1166632090078754e-06
   DEBUG:root:i=952 residual=1.116652061908381e-06
   DEBUG:root:i=953 residual=1.1166409147829771e-06
   DEBUG:root:i=954 residual=1.1166297984161713e-06
   DEBUG:root:i=955 residual=1.116618651305747e-06
   DEBUG:root:i=956 residual=1.1166075472472828e-06
   DEBUG:root:i=957 residual=1.1165964370507423e-06
   DEBUG:root:i=958 residual=1.1165852530358447e-06
   DEBUG:root:i=959 residual=1.1165741181956612e-06
   DEBUG:root:i=960 residual=1.1165630079824381e-06
   DEBUG:root:i=961 residual=1.1165518731809352e-06
   DEBUG:root:i=962 residual=1.1165407322175411e-06
   DEBUG:root:i=963 residual=1.1165296158559507e-06
   DEBUG:root:i=964 residual=1.1165184687433225e-06
   DEBUG:root:i=965 residual=1.116507346227438e-06
   DEBUG:root:i=966 residual=1.116496236022501e-06
   DEBUG:root:i=967 residual=1.116485076614236e-06
   DEBUG:root:i=968 residual=1.116473960239496e-06
   DEBUG:root:i=969 residual=1.1164628131325085e-06
   DEBUG:root:i=970 residual=1.116451702916907e-06
   DEBUG:root:i=971 residual=1.1164405681156316e-06
   DEBUG:root:i=972 residual=1.116429439453241e-06
   DEBUG:root:i=973 residual=1.1164182984869462e-06
   DEBUG:root:i=974 residual=1.1164071698221699e-06
   DEBUG:root:i=975 residual=1.1163960719179047e-06
   DEBUG:root:i=976 residual=1.116384930972681e-06
   DEBUG:root:i=977 residual=1.1163738084599879e-06
   DEBUG:root:i=978 residual=1.1163626674936766e-06
   DEBUG:root:i=979 residual=1.1163515511343637e-06
   DEBUG:root:i=980 residual=1.1163403917235397e-06
   DEBUG:root:i=981 residual=1.1163292691945861e-06
   DEBUG:root:i=982 residual=1.1163181528460491e-06
   DEBUG:root:i=983 residual=1.1163070057307069e-06
   DEBUG:root:i=984 residual=1.1162959139758056e-06
   DEBUG:root:i=985 residual=1.116284779182312e-06
   DEBUG:root:i=986 residual=1.1162736320625505e-06
   DEBUG:root:i=987 residual=1.1162624849394164e-06
   DEBUG:root:i=988 residual=1.116251362420648e-06
   DEBUG:root:i=989 residual=1.116240215310422e-06
   DEBUG:root:i=990 residual=1.1162290927866347e-06
   DEBUG:root:i=991 residual=1.1162179518281892e-06
   DEBUG:root:i=992 residual=1.1162068477672951e-06
   DEBUG:root:i=993 residual=1.1161957191227239e-06
   DEBUG:root:i=994 residual=1.1161846089133576e-06
   DEBUG:root:i=995 residual=1.1161734679654626e-06
   DEBUG:root:i=996 residual=1.1161623085395086e-06
   DEBUG:root:i=997 residual=1.116151179858536e-06
   DEBUG:root:i=998 residual=1.1161400819591827e-06
   DEBUG:root:i=999 residual=1.1161289348623947e-06
   INFO:root:rank=0 pagerank=1.0717e+01 url=www.lawfareblog.com/snowden-revelations
   INFO:root:rank=1 pagerank=1.0717e+01 url=www.lawfareblog.com/lawfare-job-board
   INFO:root:rank=2 pagerank=1.0717e+01 url=www.lawfareblog.com/masthead
   INFO:root:rank=3 pagerank=1.0717e+01 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban
   INFO:root:rank=4 pagerank=1.0717e+01 url=www.lawfareblog.com/subscribe-lawfare
   INFO:root:rank=5 pagerank=1.0717e+01 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
   INFO:root:rank=6 pagerank=1.0717e+01 url=www.lawfareblog.com/documents-related-mueller-investigation
   INFO:root:rank=7 pagerank=1.0717e+01 url=www.lawfareblog.com/our-comments-policy
   INFO:root:rank=8 pagerank=1.0717e+01 url=www.lawfareblog.com/upcoming-events
   INFO:root:rank=9 pagerank=1.0717e+01 url=www.lawfareblog.com/topics
   
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2
   DEBUG:root:computing indices
   DEBUG:root:computing values
   DEBUG:root:i=0 residual=5.117091737779263
   DEBUG:root:i=1 residual=2.98053008303595
   DEBUG:root:i=2 residual=2.449292818582698
   DEBUG:root:i=3 residual=1.6987284737084791
   DEBUG:root:i=4 residual=1.1001745208166707
   DEBUG:root:i=5 residual=0.7298141615548736
   DEBUG:root:i=6 residual=0.5148033649456959
   DEBUG:root:i=7 residual=0.379607745430235
   DEBUG:root:i=8 residual=0.28510088668996886
   DEBUG:root:i=9 residual=0.21571461966622457
   DEBUG:root:i=10 residual=0.16483486568237504
   DEBUG:root:i=11 residual=0.1284152396680088
   DEBUG:root:i=12 residual=0.1030455774372485
   DEBUG:root:i=13 residual=0.08559131776304513
   DEBUG:root:i=14 residual=0.07334083824194426
   DEBUG:root:i=15 residual=0.06422892501564025
   DEBUG:root:i=16 residual=0.05689718274094965
   DEBUG:root:i=17 residual=0.05057021765043342
   DEBUG:root:i=18 residual=0.044862600007101766
   DEBUG:root:i=19 residual=0.039611628658568994
   DEBUG:root:i=20 residual=0.0347650381074808
   DEBUG:root:i=21 residual=0.030316039460314904
   DEBUG:root:i=22 residual=0.026269787831671054
   DEBUG:root:i=23 residual=0.022628211410250083
   DEBUG:root:i=24 residual=0.019384696598280026
   DEBUG:root:i=25 residual=0.016523632996702838
   DEBUG:root:i=26 residual=0.014022056258493738
   DEBUG:root:i=27 residual=0.011851941264138494
   DEBUG:root:i=28 residual=0.00998243979204807
   DEBUG:root:i=29 residual=0.008381762116946355
   DEBUG:root:i=30 residual=0.007018614774100387
   DEBUG:root:i=31 residual=0.005863211354544911
   DEBUG:root:i=32 residual=0.004887918250714745
   DEBUG:root:i=33 residual=0.004067610600953806
   DEBUG:root:i=34 residual=0.003379811115970747
   DEBUG:root:i=35 residual=0.0028046748687071688
   DEBUG:root:i=36 residual=0.0023248713001214946
   DEBUG:root:i=37 residual=0.001925403104095444
   DEBUG:root:i=38 residual=0.001593391437515629
   DEBUG:root:i=39 residual=0.00131784844916224
   DEBUG:root:i=40 residual=0.0010894514370479544
   DEBUG:root:i=41 residual=0.000900327846340177
   DEBUG:root:i=42 residual=0.0007438565564460495
   DEBUG:root:i=43 residual=0.0006144882151818832
   DEBUG:root:i=44 residual=0.0005075855213050842
   DEBUG:root:i=45 residual=0.00041928312983478697
   DEBUG:root:i=46 residual=0.00034636609086266757
   DEBUG:root:i=47 residual=0.0002861653004752691
   DEBUG:root:i=48 residual=0.00023646824089293243
   DEBUG:root:i=49 residual=0.0001954432403252256
   DEBUG:root:i=50 residual=0.0001615755358105362
   DEBUG:root:i=51 residual=0.0001336135346778768
   DEBUG:root:i=52 residual=0.00011052381444720629
   DEBUG:root:i=53 residual=9.145355812181814e-05
   DEBUG:root:i=54 residual=7.569927948217119e-05
   DEBUG:root:i=55 residual=6.268084359453943e-05
   DEBUG:root:i=56 residual=5.191992682084757e-05
   DEBUG:root:i=57 residual=4.302218599272531e-05
   DEBUG:root:i=58 residual=3.566251753521154e-05
   DEBUG:root:i=59 residual=2.9572884287826355e-05
   DEBUG:root:i=60 residual=2.4532271635748374e-05
   DEBUG:root:i=61 residual=2.0358406299235085e-05
   DEBUG:root:i=62 residual=1.690093215783077e-05
   DEBUG:root:i=63 residual=1.4035789014296522e-05
   DEBUG:root:i=64 residual=1.1660583554632329e-05
   DEBUG:root:i=65 residual=9.690778025185849e-06
   DEBUG:root:i=66 residual=8.056552428997387e-06
   DEBUG:root:i=67 residual=6.700221198643203e-06
   DEBUG:root:i=68 residual=5.5741062051091394e-06
   DEBUG:root:i=69 residual=4.638785239908982e-06
   DEBUG:root:i=70 residual=3.8616494126696716e-06
   DEBUG:root:i=71 residual=3.215714683960239e-06
   DEBUG:root:i=72 residual=2.6786424938967927e-06
   DEBUG:root:i=73 residual=2.2319324385060614e-06
   DEBUG:root:i=74 residual=1.8602565405583407e-06
   DEBUG:root:i=75 residual=1.5509100783357916e-06
   DEBUG:root:i=76 residual=1.2933583897149281e-06
   DEBUG:root:i=77 residual=1.0788627264271694e-06
   DEBUG:root:i=78 residual=9.00171242964162e-07
   INFO:root:rank=0 pagerank=4.6096e+00 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
   INFO:root:rank=1 pagerank=2.9870e+00 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
   INFO:root:rank=2 pagerank=2.9672e+00 url=www.lawfareblog.com/opening-statement-david-holmes
   INFO:root:rank=3 pagerank=2.0175e+00 url=www.lawfareblog.com/senate-examines-threats-homeland
   INFO:root:rank=4 pagerank=1.8771e+00 url=www.lawfareblog.com/what-make-first-day-impeachment-hearings
   INFO:root:rank=5 pagerank=1.8764e+00 url=www.lawfareblog.com/livestream-house-armed-services-committee-hearing-f-35-program
   INFO:root:rank=6 pagerank=1.8695e+00 url=www.lawfareblog.com/whats-house-resolution-impeachment
   INFO:root:rank=7 pagerank=1.7657e+00 url=www.lawfareblog.com/congress-us-policy-toward-syria-and-turkey-overview-recent-hearings
   INFO:root:rank=8 pagerank=1.6809e+00 url=www.lawfareblog.com/summary-david-holmess-deposition-testimony
   INFO:root:rank=9 pagerank=9.8355e-01 url=www.lawfareblog.com/events
   
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2 --alpha=0.99999
   DEBUG:root:computing indices
   DEBUG:root:computing values
   DEBUG:root:i=0 residual=6.020047158598338
   DEBUG:root:i=1 residual=4.125217441769446
   DEBUG:root:i=2 residual=3.9881363009869197
   DEBUG:root:i=3 residual=3.2540834140553874
   DEBUG:root:i=4 residual=2.479371048853352
   DEBUG:root:i=5 residual=1.9349419151783867
   DEBUG:root:i=6 residual=1.6057343341065766
   DEBUG:root:i=7 residual=1.3929795594701166
   DEBUG:root:i=8 residual=1.2307955278545306
   DEBUG:root:i=9 residual=1.0955817359281075
   DEBUG:root:i=10 residual=0.9849005333356389
   DEBUG:root:i=11 residual=0.9026889316556481
   DEBUG:root:i=12 residual=0.852175549459687
   DEBUG:root:i=13 residual=0.8327355848363157
   DEBUG:root:i=14 residual=0.8394605543654504
   DEBUG:root:i=15 residual=0.8648911422928319
   DEBUG:root:i=16 residual=0.9013583460852844
   DEBUG:root:i=17 residual=0.9424910195917103
   DEBUG:root:i=18 residual=0.9836540795717678
   DEBUG:root:i=19 residual=1.0217771163665068
   DEBUG:root:i=20 residual=1.054998115348103
   DEBUG:root:i=21 residual=1.0823231164271403
   DEBUG:root:i=22 residual=1.1033581988344783
   DEBUG:root:i=23 residual=1.1181129252204807
   DEBUG:root:i=24 residual=1.1268598573407098
   DEBUG:root:i=25 residual=1.1300349445777367
   DEBUG:root:i=26 residual=1.12816711866896
   DEBUG:root:i=27 residual=1.121828792091958
   DEBUG:root:i=28 residual=1.1116014493254107
   DEBUG:root:i=29 residual=1.0980522302304485
   DEBUG:root:i=30 residual=1.0817185596774703
   DEBUG:root:i=31 residual=1.0630986669910993
   DEBUG:root:i=32 residual=1.0426463921315445
   DEBUG:root:i=33 residual=1.0207690746086773
   DEBUG:root:i=34 residual=0.9978276164890215
   DEBUG:root:i=35 residual=0.9741380338898341
   DEBUG:root:i=36 residual=0.9499739820415545
   DEBUG:root:i=37 residual=0.9255698705755535
   DEBUG:root:i=38 residual=0.9011242873406821
   DEBUG:root:i=39 residual=0.8768035274332444
   DEBUG:root:i=40 residual=0.8527450842663781
   DEBUG:root:i=41 residual=0.8290610052992181
   DEBUG:root:i=42 residual=0.8058410495672168
   DEBUG:root:i=43 residual=0.7831556098479961
   DEBUG:root:i=44 residual=0.761058381116677
   DEBUG:root:i=45 residual=0.7395887704505432
   DEBUG:root:i=46 residual=0.7187740529771973
   DEBUG:root:i=47 residual=0.6986312848034677
   DEBUG:root:i=48 residual=0.6791689878884487
   DEBUG:root:i=49 residual=0.660388624135377
   DEBUG:root:i=50 residual=0.6422858770413923
   DEBUG:root:i=51 residual=0.624851759420584
   DEBUG:root:i=52 residual=0.6080735652788183
   DEBUG:root:i=53 residual=0.5919356830726986
   DEBUG:root:i=54 residual=0.5764202864860514
   DEBUG:root:i=55 residual=0.5615079176157761
   DEBUG:root:i=56 residual=0.5471779761581916
   DEBUG:root:i=57 residual=0.533409126883692
   DEBUG:root:i=58 residual=0.5201796364213006
   DEBUG:root:i=59 residual=0.5074676491716834
   DEBUG:root:i=60 residual=0.49525141104318987
   DEBUG:root:i=61 residual=0.4835094486687584
   DEBUG:root:i=62 residual=0.4722207108158496
   DEBUG:root:i=63 residual=0.4613646778464621
   DEBUG:root:i=64 residual=0.45092144431600734
   DEBUG:root:i=65 residual=0.4408717791154567
   DEBUG:root:i=66 residual=0.4311971669535066
   DEBUG:root:i=67 residual=0.4218798344392718
   DEBUG:root:i=68 residual=0.41290276355526023
   DEBUG:root:i=69 residual=0.40424969489794726
   DEBUG:root:i=70 residual=0.39590512270430944
   DEBUG:root:i=71 residual=0.38785428337036426
   DEBUG:root:i=72 residual=0.3800831388980651
   DEBUG:root:i=73 residual=0.3725783564736709
   DEBUG:root:i=74 residual=0.36532728518080587
   DEBUG:root:i=75 residual=0.3583179306793046
   DEBUG:root:i=76 residual=0.3515389285348547
   DEBUG:root:i=77 residual=0.3449795167592671
   DEBUG:root:i=78 residual=0.3386295080155802
   DEBUG:root:i=79 residual=0.3324792618523278
   DEBUG:root:i=80 residual=0.3265196572563075
   DEBUG:root:i=81 residual=0.32074206574965036
   DEBUG:root:i=82 residual=0.3151383252042338
   DEBUG:root:i=83 residual=0.3097007145026567
   DEBUG:root:i=84 residual=0.30442192913835386
   DEBUG:root:i=85 residual=0.29929505781827936
   DEBUG:root:i=86 residual=0.294313560106377
   DEBUG:root:i=87 residual=0.2894712451269959
   DEBUG:root:i=88 residual=0.284762251331256
   DEBUG:root:i=89 residual=0.280181027317164
   DEBUG:root:i=90 residual=0.27572231368383654
   DEBUG:root:i=91 residual=0.27138112589359337
   DEBUG:root:i=92 residual=0.26715273810939894
   DEBUG:root:i=93 residual=0.26303266797106234
   DEBUG:root:i=94 residual=0.2590166622715873
   DEBUG:root:i=95 residual=0.2551006834921426
   DEBUG:root:i=96 residual=0.2512808971544273
   DEBUG:root:i=97 residual=0.2475536599481658
   DEBUG:root:i=98 residual=0.24391550859207659
   DEBUG:root:i=99 residual=0.24036314938765183
   DEBUG:root:i=100 residual=0.23689344842541657
   DEBUG:root:i=101 residual=0.2335034224054999
   DEBUG:root:i=102 residual=0.23019023003522826
   DEBUG:root:i=103 residual=0.2269511639681057
   DEBUG:root:i=104 residual=0.22378364325057895
   DEBUG:root:i=105 residual=0.22068520624384283
   DEBUG:root:i=106 residual=0.21765350399099503
   DEBUG:root:i=107 residual=0.2146862939997152
   DEBUG:root:i=108 residual=0.21178143441439734
   DEBUG:root:i=109 residual=0.2089368785510989
   DEBUG:root:i=110 residual=0.20615066977226762
   DEBUG:root:i=111 residual=0.20342093667799774
   DEBUG:root:i=112 residual=0.20074588859344278
   DEBUG:root:i=113 residual=0.1981238113318517
   DEBUG:root:i=114 residual=0.19555306321580868
   DEBUG:root:i=115 residual=0.1930320713386535
   DEBUG:root:i=116 residual=0.190559328050549
   DEBUG:root:i=117 residual=0.18813338765408583
   DEBUG:root:i=118 residual=0.18575286329555035
   DEBUG:root:i=119 residual=0.1834164240390443
   DEBUG:root:i=120 residual=0.1811227921109729
   DEBUG:root:i=121 residual=0.17887074030420047
   DEBUG:root:i=122 residual=0.17665908953118137
   DEBUG:root:i=123 residual=0.17448670651635872
   DEBUG:root:i=124 residual=0.1723525016187073
   DEBUG:root:i=125 residual=0.17025542677634212
   DEBUG:root:i=126 residual=0.16819447356508088
   DEBUG:root:i=127 residual=0.16616867136374217
   DEBUG:root:i=128 residual=0.16417708561953254
   DEBUG:root:i=129 residual=0.16221881620724704
   DEBUG:root:i=130 residual=0.1602929958761691
   DEBUG:root:i=131 residual=0.1583987887796131
   DEBUG:root:i=132 residual=0.15653538908174722
   DEBUG:root:i=133 residual=0.1547020196369984
   DEBUG:root:i=134 residual=0.1528979307379199
   DEBUG:root:i=135 residual=0.15112239892701082
   DEBUG:root:i=136 residual=0.14937472586902895
   DEBUG:root:i=137 residual=0.14765423727993218
   DEBUG:root:i=138 residual=0.14596028190943355
   DEBUG:root:i=139 residual=0.144292230573783
   DEBUG:root:i=140 residual=0.14264947523610214
   DEBUG:root:i=141 residual=0.14103142813137234
   DEBUG:root:i=142 residual=0.1394375209337149
   DEBUG:root:i=143 residual=0.1378672039635581
   DEBUG:root:i=144 residual=0.1363199454322788
   DEBUG:root:i=145 residual=0.1347952307226814
   DEBUG:root:i=146 residual=0.13329256170283701
   DEBUG:root:i=147 residual=0.13181145607196312
   DEBUG:root:i=148 residual=0.1303514467362637
   DEBUG:root:i=149 residual=0.1289120812133364
   DEBUG:root:i=150 residual=0.12749292106360582
   DEBUG:root:i=151 residual=0.12609354134734885
   DEBUG:root:i=152 residual=0.12471353010594777
   DEBUG:root:i=153 residual=0.12335248786627648
   DEBUG:root:i=154 residual=0.12201002716682151
   DEBUG:root:i=155 residual=0.12068577210462504
   DEBUG:root:i=156 residual=0.11937935790204376
   DEBUG:root:i=157 residual=0.11809043049204891
   DEBUG:root:i=158 residual=0.11681864612142909
   DEBUG:root:i=159 residual=0.11556367097105147
   DEBUG:root:i=160 residual=0.11432518079204021
   DEBUG:root:i=161 residual=0.11310286055750852
   DEBUG:root:i=162 residual=0.11189640412876103
   DEBUG:root:i=163 residual=0.11070551393540295
   DEBUG:root:i=164 residual=0.10952990066893435
   DEBUG:root:i=165 residual=0.10836928298873785
   DEBUG:root:i=166 residual=0.10722338724018766
   DEBUG:root:i=167 residual=0.10609194718440836
   DEBUG:root:i=168 residual=0.10497470373884166
   DEBUG:root:i=169 residual=0.10387140472834207
   DEBUG:root:i=170 residual=0.10278180464630424
   DEBUG:root:i=171 residual=0.10170566442535023
   DEBUG:root:i=172 residual=0.10064275121705465
   DEBUG:root:i=173 residual=0.0995928381804941
   DEBUG:root:i=174 residual=0.09855570427909105
   DEBUG:root:i=175 residual=0.09753113408541585
   DEBUG:root:i=176 residual=0.09651891759360272
   DEBUG:root:i=177 residual=0.09551885003913235
   DEBUG:root:i=178 residual=0.09453073172545062
   DEBUG:root:i=179 residual=0.09355436785739958
   DEBUG:root:i=180 residual=0.0925895683809876
   DEBUG:root:i=181 residual=0.09163614782921543
   DEBUG:root:i=182 residual=0.09069392517388265
   DEBUG:root:i=183 residual=0.0897627236828855
   DEBUG:root:i=184 residual=0.08884237078298549
   DEBUG:root:i=185 residual=0.08793269792765579
   DEBUG:root:i=186 residual=0.08703354046989757
   DEBUG:root:i=187 residual=0.08614473753984857
   DEBUG:root:i=188 residual=0.0852661319267636
   DEBUG:root:i=189 residual=0.08439756996554358
   DEBUG:root:i=190 residual=0.08353890142733975
   DEBUG:root:i=191 residual=0.08268997941406955
   DEBUG:root:i=192 residual=0.08185066025700558
   DEBUG:root:i=193 residual=0.08102080341882723
   DEBUG:root:i=194 residual=0.08020027139934734
   DEBUG:root:i=195 residual=0.07938892964453952
   DEBUG:root:i=196 residual=0.07858664645890676
   DEBUG:root:i=197 residual=0.07779329292095676
   DEBUG:root:i=198 residual=0.07700874280164458
   DEBUG:root:i=199 residual=0.07623287248571617
   DEBUG:root:i=200 residual=0.07546556089591178
   DEBUG:root:i=201 residual=0.0747066894196823
   DEBUG:root:i=202 residual=0.0739561418385939
   DEBUG:root:i=203 residual=0.07321380426017265
   DEBUG:root:i=204 residual=0.07247956505200123
   DEBUG:root:i=205 residual=0.07175331477827437
   DEBUG:root:i=206 residual=0.07103494613841879
   DEBUG:root:i=207 residual=0.07032435390778148
   DEBUG:root:i=208 residual=0.06962143488054548
   DEBUG:root:i=209 residual=0.06892608781424504
   DEBUG:root:i=210 residual=0.06823821337658947
   DEBUG:root:i=211 residual=0.06755771409360994
   DEBUG:root:i=212 residual=0.06688449429998065
   DEBUG:root:i=213 residual=0.06621846009073355
   DEBUG:root:i=214 residual=0.06555951927458538
   DEBUG:root:i=215 residual=0.06490758132898311
   DEBUG:root:i=216 residual=0.06426255735648424
   DEBUG:root:i=217 residual=0.06362436004257929
   DEBUG:root:i=218 residual=0.06299290361501016
   DEBUG:root:i=219 residual=0.06236810380423436
   DEBUG:root:i=220 residual=0.06174987780534904
   DEBUG:root:i=221 residual=0.0611381442410704
   DEBUG:root:i=222 residual=0.060532823126132104
   DEBUG:root:i=223 residual=0.059933835832586144
   DEBUG:root:i=224 residual=0.05934110505626786
   DEBUG:root:i=225 residual=0.05875455478452219
   DEBUG:root:i=226 residual=0.058174110264627064
   DEBUG:root:i=227 residual=0.05759969797350652
   DEBUG:root:i=228 residual=0.05703124558816477
   DEBUG:root:i=229 residual=0.05646868195716266
   DEBUG:root:i=230 residual=0.05591193707300105
   DEBUG:root:i=231 residual=0.055360942045208035
   DEBUG:root:i=232 residual=0.05481562907440927
   DEBUG:root:i=233 residual=0.05427593142718034
   DEBUG:root:i=234 residual=0.053741783411473266
   DEBUG:root:i=235 residual=0.0532131203530929
   DEBUG:root:i=236 residual=0.052689878572547255
   DEBUG:root:i=237 residual=0.05217199536291101
   DEBUG:root:i=238 residual=0.05165940896810442
   DEBUG:root:i=239 residual=0.05115205856191585
   DEBUG:root:i=240 residual=0.05064988422765727
   DEBUG:root:i=241 residual=0.05015282693846438
   DEBUG:root:i=242 residual=0.04966082853791944
   DEBUG:root:i=243 residual=0.049173831721617225
   DEBUG:root:i=244 residual=0.04869178001898708
   DEBUG:root:i=245 residual=0.0482146177756754
   DEBUG:root:i=246 residual=0.04774229013655667
   DEBUG:root:i=247 residual=0.04727474302910051
   DEBUG:root:i=248 residual=0.04681192314727954
   DEBUG:root:i=249 residual=0.046353777935877256
   DEBUG:root:i=250 residual=0.04590025557529836
   DEBUG:root:i=251 residual=0.04545130496675755
   DEBUG:root:i=252 residual=0.04500687571788834
   DEBUG:root:i=253 residual=0.044566918128726714
   DEBUG:root:i=254 residual=0.044131383178147424
   DEBUG:root:i=255 residual=0.04370022251061449
   DEBUG:root:i=256 residual=0.043273388423224576
   DEBUG:root:i=257 residual=0.042850833853311586
   DEBUG:root:i=258 residual=0.0424325123661044
   DEBUG:root:i=259 residual=0.04201837814298317
   DEBUG:root:i=260 residual=0.04160838596981976
   DEBUG:root:i=261 residual=0.041202491225738394
   DEBUG:root:i=262 residual=0.04080064987220896
   DEBUG:root:i=263 residual=0.040402818442346146
   DEBUG:root:i=264 residual=0.040008954030446506
   DEBUG:root:i=265 residual=0.039619014281962274
   DEBUG:root:i=266 residual=0.039232957383520325
   DEBUG:root:i=267 residual=0.03885074205336806
   DEBUG:root:i=268 residual=0.03847232753199642
   DEBUG:root:i=269 residual=0.03809767357293068
   DEBUG:root:i=270 residual=0.03772674043383355
   DEBUG:root:i=271 residual=0.03735948886787884
   DEBUG:root:i=272 residual=0.03699588011511072
   DEBUG:root:i=273 residual=0.03663587589437928
   DEBUG:root:i=274 residual=0.03627943839506192
   DEBUG:root:i=275 residual=0.03592653026925883
   DEBUG:root:i=276 residual=0.03557711462418702
   DEBUG:root:i=277 residual=0.0352311550144844
   DEBUG:root:i=278 residual=0.034888615435051666
   DEBUG:root:i=279 residual=0.03454946031379912
   DEBUG:root:i=280 residual=0.0342136545046654
   DEBUG:root:i=281 residual=0.03388116328078124
   DEBUG:root:i=282 residual=0.033551952327847766
   DEBUG:root:i=283 residual=0.033225987737533796
   DEBUG:root:i=284 residual=0.03290323600117859
   DEBUG:root:i=285 residual=0.03258366400346022
   DEBUG:root:i=286 residual=0.03226723901642335
   DEBUG:root:i=287 residual=0.03195392869347637
   DEBUG:root:i=288 residual=0.03164370106350451
   DEBUG:root:i=289 residual=0.03133652452530321
   DEBUG:root:i=290 residual=0.03103236784183417
   DEBUG:root:i=291 residual=0.030731200134958756
   DEBUG:root:i=292 residual=0.030432990879958086
   DEBUG:root:i=293 residual=0.030137709900418187
   DEBUG:root:i=294 residual=0.02984532736302683
   DEBUG:root:i=295 residual=0.02955581377262362
   DEBUG:root:i=296 residual=0.02926913996731944
   DEBUG:root:i=297 residual=0.028985277113640892
   DEBUG:root:i=298 residual=0.028704196701850336
   DEBUG:root:i=299 residual=0.028425870541416126
   DEBUG:root:i=300 residual=0.028150270756363734
   DEBUG:root:i=301 residual=0.027877369781013617
   DEBUG:root:i=302 residual=0.027607140355483447
   DEBUG:root:i=303 residual=0.027339555521617414
   DEBUG:root:i=304 residual=0.027074588618715364
   DEBUG:root:i=305 residual=0.02681221327945842
   DEBUG:root:i=306 residual=0.026552403425940185
   DEBUG:root:i=307 residual=0.02629513326577177
   DEBUG:root:i=308 residual=0.026040377288131206
   DEBUG:root:i=309 residual=0.025788110260093724
   DEBUG:root:i=310 residual=0.02553830722288957
   DEBUG:root:i=311 residual=0.02529094348827121
   DEBUG:root:i=312 residual=0.025045994634907996
   DEBUG:root:i=313 residual=0.024803436504952748
   DEBUG:root:i=314 residual=0.024563245200579734
   DEBUG:root:i=315 residual=0.024325397080585492
   DEBUG:root:i=316 residual=0.024089868757052914
   DEBUG:root:i=317 residual=0.02385663709224167
   DEBUG:root:i=318 residual=0.023625679195193334
   DEBUG:root:i=319 residual=0.023396972418717958
   DEBUG:root:i=320 residual=0.02317049435628494
   DEBUG:root:i=321 residual=0.022946222838969448
   DEBUG:root:i=322 residual=0.02272413593250177
   DEBUG:root:i=323 residual=0.0225042119343299
   DEBUG:root:i=324 residual=0.022286429370695804
   DEBUG:root:i=325 residual=0.022070766993925316
   DEBUG:root:i=326 residual=0.02185720377949335
   DEBUG:root:i=327 residual=0.021645718923427877
   DEBUG:root:i=328 residual=0.021436291839486556
   DEBUG:root:i=329 residual=0.021228902156664693
   DEBUG:root:i=330 residual=0.02102352971646251
   DEBUG:root:i=331 residual=0.020820154570387647
   DEBUG:root:i=332 residual=0.020618756977393723
   DEBUG:root:i=333 residual=0.020419317401478804
   DEBUG:root:i=334 residual=0.020221816509147117
   DEBUG:root:i=335 residual=0.020026235167125487
   DEBUG:root:i=336 residual=0.019832554439862243
   DEBUG:root:i=337 residual=0.01964075558731007
   DEBUG:root:i=338 residual=0.019450820062553685
   DEBUG:root:i=339 residual=0.019262729509687517
   DEBUG:root:i=340 residual=0.019076465761426062
   DEBUG:root:i=341 residual=0.018892010837023385
   DEBUG:root:i=342 residual=0.018709346940095906
   DEBUG:root:i=343 residual=0.018528456456483895
   DEBUG:root:i=344 residual=0.01834932195218048
   DEBUG:root:i=345 residual=0.018171926171287597
   DEBUG:root:i=346 residual=0.017996252033896935
   DEBUG:root:i=347 residual=0.017822282634260125
   DEBUG:root:i=348 residual=0.017650001238616375
   DEBUG:root:i=349 residual=0.017479391283413852
   DEBUG:root:i=350 residual=0.01731043637331489
   DEBUG:root:i=351 residual=0.017143120279292656
   DEBUG:root:i=352 residual=0.01697742693685791
   DEBUG:root:i=353 residual=0.01681334044411293
   DEBUG:root:i=354 residual=0.016650845060030055
   DEBUG:root:i=355 residual=0.016489925202643287
   DEBUG:root:i=356 residual=0.016330565447260345
   DEBUG:root:i=357 residual=0.016172750524748893
   DEBUG:root:i=358 residual=0.016016465319877812
   DEBUG:root:i=359 residual=0.015861694869487027
   DEBUG:root:i=360 residual=0.015708424361001536
   DEBUG:root:i=361 residual=0.015556639130642931
   DEBUG:root:i=362 residual=0.015406324661890045
   DEBUG:root:i=363 residual=0.015257466583854316
   DEBUG:root:i=364 residual=0.015110050669682788
   DEBUG:root:i=365 residual=0.014964062835056628
   DEBUG:root:i=366 residual=0.01481948913659724
   DEBUG:root:i=367 residual=0.014676315770355641
   DEBUG:root:i=368 residual=0.01453452907035622
   DEBUG:root:i=369 residual=0.014394115507059324
   DEBUG:root:i=370 residual=0.014255061685984567
   DEBUG:root:i=371 residual=0.014117354346163962
   DEBUG:root:i=372 residual=0.013980980358826741
   DEBUG:root:i=373 residual=0.013845926725902112
   DEBUG:root:i=374 residual=0.013712180578695615
   DEBUG:root:i=375 residual=0.013579729176491069
   DEBUG:root:i=376 residual=0.013448559905189129
   DEBUG:root:i=377 residual=0.013318660276037838
   DEBUG:root:i=378 residual=0.013190017924228003
   DEBUG:root:i=379 residual=0.013062620607621245
   DEBUG:root:i=380 residual=0.012936456205473056
   DEBUG:root:i=381 residual=0.012811512717180657
   DEBUG:root:i=382 residual=0.012687778260940946
   DEBUG:root:i=383 residual=0.012565241072659583
   DEBUG:root:i=384 residual=0.01244388950458042
   DEBUG:root:i=385 residual=0.012323712024119509
   DEBUG:root:i=386 residual=0.01220469721275291
   DEBUG:root:i=387 residual=0.012086833764666524
   DEBUG:root:i=388 residual=0.011970110485753553
   DEBUG:root:i=389 residual=0.01185451629234772
   DEBUG:root:i=390 residual=0.01174004021016328
   DEBUG:root:i=391 residual=0.011626671373102911
   DEBUG:root:i=392 residual=0.011514399022130755
   DEBUG:root:i=393 residual=0.011403212504320135
   DEBUG:root:i=394 residual=0.0112931012715472
   DEBUG:root:i=395 residual=0.0111840548795504
   DEBUG:root:i=396 residual=0.011076062986853355
   DEBUG:root:i=397 residual=0.010969115353684602
   DEBUG:root:i=398 residual=0.010863201840955124
   DEBUG:root:i=399 residual=0.01075831240922798
   DEBUG:root:i=400 residual=0.010654437117673067
   DEBUG:root:i=401 residual=0.010551566123143835
   DEBUG:root:i=402 residual=0.010449689679065996
   DEBUG:root:i=403 residual=0.01034879813456374
   DEBUG:root:i=404 residual=0.010248881933422306
   DEBUG:root:i=405 residual=0.010149931613157043
   DEBUG:root:i=406 residual=0.01005193780406477
   DEBUG:root:i=407 residual=0.009954891228264724
   DEBUG:root:i=408 residual=0.009858782698804034
   DEBUG:root:i=409 residual=0.009763603118636983
   DEBUG:root:i=410 residual=0.009669343479915795
   DEBUG:root:i=411 residual=0.009575994862884166
   DEBUG:root:i=412 residual=0.00948354843512956
   DEBUG:root:i=413 residual=0.00939199545061883
   DEBUG:root:i=414 residual=0.009301327248898119
   DEBUG:root:i=415 residual=0.009211535254139092
   DEBUG:root:i=416 residual=0.009122610974408448
   DEBUG:root:i=417 residual=0.009034546000782007
   DEBUG:root:i=418 residual=0.008947332006407023
   DEBUG:root:i=419 residual=0.008860960745865674
   DEBUG:root:i=420 residual=0.008775424054226901
   DEBUG:root:i=421 residual=0.008690713846266237
   DEBUG:root:i=422 residual=0.008606822115683995
   DEBUG:root:i=423 residual=0.008523740934368945
   DEBUG:root:i=424 residual=0.008441462451506204
   DEBUG:root:i=425 residual=0.008359978892884989
   DEBUG:root:i=426 residual=0.008279282560139605
   DEBUG:root:i=427 residual=0.00819936582991112
   DEBUG:root:i=428 residual=0.008120221153225764
   DEBUG:root:i=429 residual=0.008041841054648408
   DEBUG:root:i=430 residual=0.00796421813162199
   DEBUG:root:i=431 residual=0.00788734505366078
   DEBUG:root:i=432 residual=0.007811214561773858
   DEBUG:root:i=433 residual=0.007735819467601035
   DEBUG:root:i=434 residual=0.007661152652801774
   DEBUG:root:i=435 residual=0.007587207068343936
   DEBUG:root:i=436 residual=0.007513975733820419
   DEBUG:root:i=437 residual=0.007441451736748017
   DEBUG:root:i=438 residual=0.00736962823191828
   DEBUG:root:i=439 residual=0.007298498440732615
   DEBUG:root:i=440 residual=0.007228055650524666
   DEBUG:root:i=441 residual=0.007158293213908545
   DEBUG:root:i=442 residual=0.007089204548201688
   DEBUG:root:i=443 residual=0.0070207831346371955
   DEBUG:root:i=444 residual=0.006953022517917487
   DEBUG:root:i=445 residual=0.006885916305428398
   DEBUG:root:i=446 residual=0.00681945816676448
   DEBUG:root:i=447 residual=0.006753641832933646
   DEBUG:root:i=448 residual=0.0066884610959704206
   DEBUG:root:i=449 residual=0.006623909808151921
   DEBUG:root:i=450 residual=0.006559981881514039
   DEBUG:root:i=451 residual=0.006496671287193837
   DEBUG:root:i=452 residual=0.006433972054909291
   DEBUG:root:i=453 residual=0.006371878272344331
   DEBUG:root:i=454 residual=0.006310384084605647
   DEBUG:root:i=455 residual=0.006249483693623492
   DEBUG:root:i=456 residual=0.006189171357588678
   DEBUG:root:i=457 residual=0.006129441390505328
   DEBUG:root:i=458 residual=0.0060702881614875
   DEBUG:root:i=459 residual=0.006011706094332376
   DEBUG:root:i=460 residual=0.005953689666904175
   DEBUG:root:i=461 residual=0.0058962334106792144
   DEBUG:root:i=462 residual=0.005839331910198687
   DEBUG:root:i=463 residual=0.005782979802512162
   DEBUG:root:i=464 residual=0.0057271717766796265
   DEBUG:root:i=465 residual=0.0056719025732976085
   DEBUG:root:i=466 residual=0.005617166983934786
   DEBUG:root:i=467 residual=0.005562959850701826
   DEBUG:root:i=468 residual=0.005509276065677758
   DEBUG:root:i=469 residual=0.005456110570524476
   DEBUG:root:i=470 residual=0.005403458355864839
   DEBUG:root:i=471 residual=0.005351314460927459
   DEBUG:root:i=472 residual=0.005299673973016916
   DEBUG:root:i=473 residual=0.005248532027061994
   DEBUG:root:i=474 residual=0.00519788380512131
   DEBUG:root:i=475 residual=0.005147724535941023
   DEBUG:root:i=476 residual=0.005098049494504423
   DEBUG:root:i=477 residual=0.005048854001620883
   DEBUG:root:i=478 residual=0.005000133423328623
   DEBUG:root:i=479 residual=0.004951883170674091
   DEBUG:root:i=480 residual=0.004904098699062236
   DEBUG:root:i=481 residual=0.004856775507968307
   DEBUG:root:i=482 residual=0.004809909140453528
   DEBUG:root:i=483 residual=0.0047634951827364306
   DEBUG:root:i=484 residual=0.004717529263767719
   DEBUG:root:i=485 residual=0.004672007054879632
   DEBUG:root:i=486 residual=0.004626924269249816
   DEBUG:root:i=487 residual=0.0045822766615944855
   DEBUG:root:i=488 residual=0.004538060027749545
   DEBUG:root:i=489 residual=0.004494270204213551
   DEBUG:root:i=490 residual=0.004450903067836598
   DEBUG:root:i=491 residual=0.0044079545353587195
   DEBUG:root:i=492 residual=0.004365420563016056
   DEBUG:root:i=493 residual=0.004323297146250674
   DEBUG:root:i=494 residual=0.004281580319214528
   DEBUG:root:i=495 residual=0.004240266154445747
   DEBUG:root:i=496 residual=0.004199350762529208
   DEBUG:root:i=497 residual=0.004158830291636125
   DEBUG:root:i=498 residual=0.004118700927230147
   DEBUG:root:i=499 residual=0.004078958891717783
   DEBUG:root:i=500 residual=0.00403960044400964
   DEBUG:root:i=501 residual=0.004000621879264468
   DEBUG:root:i=502 residual=0.003962019528410523
   DEBUG:root:i=503 residual=0.003923789757939544
   DEBUG:root:i=504 residual=0.0038859289694624546
   DEBUG:root:i=505 residual=0.00384843359943527
   DEBUG:root:i=506 residual=0.0038113001187158432
   DEBUG:root:i=507 residual=0.003774525032386921
   DEBUG:root:i=508 residual=0.003738104879245005
   DEBUG:root:i=509 residual=0.003702036231644281
   DEBUG:root:i=510 residual=0.0036663156950481755
   DEBUG:root:i=511 residual=0.0036309399077420305
   DEBUG:root:i=512 residual=0.0035959055405468993
   DEBUG:root:i=513 residual=0.00356120929646351
   DEBUG:root:i=514 residual=0.0035268479103957784
   DEBUG:root:i=515 residual=0.0034928181487819923
   DEBUG:root:i=516 residual=0.003459116809351322
   DEBUG:root:i=517 residual=0.003425740720843728
   DEBUG:root:i=518 residual=0.003392686742557261
   DEBUG:root:i=519 residual=0.0033599517642289765
   DEBUG:root:i=520 residual=0.0033275327056611066
   DEBUG:root:i=521 residual=0.003295426516415474
   DEBUG:root:i=522 residual=0.003263630175568738
   DEBUG:root:i=523 residual=0.0032321406913775465
   DEBUG:root:i=524 residual=0.0032009551010030585
   DEBUG:root:i=525 residual=0.0031700704703053863
   DEBUG:root:i=526 residual=0.003139483893474732
   DEBUG:root:i=527 residual=0.0031091924927671736
   DEBUG:root:i=528 residual=0.0030791934182939204
   DEBUG:root:i=529 residual=0.0030494838476918676
   DEBUG:root:i=530 residual=0.0030200609858849824
   DEBUG:root:i=531 residual=0.002990922064804608
   DEBUG:root:i=532 residual=0.002962064343130109
   DEBUG:root:i=533 residual=0.002933485106054311
   DEBUG:root:i=534 residual=0.002905181664980373
   DEBUG:root:i=535 residual=0.0028771513573171313
   DEBUG:root:i=536 residual=0.0028493915461996642
   DEBUG:root:i=537 residual=0.002821899620217524
   DEBUG:root:i=538 residual=0.002794672993252822
   DEBUG:root:i=539 residual=0.0027677091041249523
   DEBUG:root:i=540 residual=0.0027410054164342047
   DEBUG:root:i=541 residual=0.0027145594182459002
   DEBUG:root:i=542 residual=0.002688368621954021
   DEBUG:root:i=543 residual=0.002662430563950316
   DEBUG:root:i=544 residual=0.002636742804430432
   DEBUG:root:i=545 residual=0.0026113029271845147
   DEBUG:root:i=546 residual=0.0025861085393406626
   DEBUG:root:i=547 residual=0.002561157271126374
   DEBUG:root:i=548 residual=0.002536446775680225
   DEBUG:root:i=549 residual=0.0025119747288293226
   DEBUG:root:i=550 residual=0.002487738828816996
   DEBUG:root:i=551 residual=0.0024637367961488793
   DEBUG:root:i=552 residual=0.0024399663733367326
   DEBUG:root:i=553 residual=0.0024164253247291488
   DEBUG:root:i=554 residual=0.0023931114362355796
   DEBUG:root:i=555 residual=0.002370022515173631
   DEBUG:root:i=556 residual=0.0023471563900454365
   DEBUG:root:i=557 residual=0.002324510910307356
   DEBUG:root:i=558 residual=0.002302083946201284
   DEBUG:root:i=559 residual=0.002279873388542826
   DEBUG:root:i=560 residual=0.0022578771485339583
   DEBUG:root:i=561 residual=0.002236093157522452
   DEBUG:root:i=562 residual=0.002214519366863012
   DEBUG:root:i=563 residual=0.002193153747685968
   DEBUG:root:i=564 residual=0.0021719942906923972
   DEBUG:root:i=565 residual=0.002151039006038223
   DEBUG:root:i=566 residual=0.0021302859230719327
   DEBUG:root:i=567 residual=0.0021097330901897523
   DEBUG:root:i=568 residual=0.0020893785745914914
   DEBUG:root:i=569 residual=0.0020692204622019582
   DEBUG:root:i=570 residual=0.002049256857390943
   DEBUG:root:i=571 residual=0.0020294858828660165
   DEBUG:root:i=572 residual=0.002009905679441453
   DEBUG:root:i=573 residual=0.0019905144058730254
   DEBUG:root:i=574 residual=0.0019713102387299702
   DEBUG:root:i=575 residual=0.0019522913721796774
   DEBUG:root:i=576 residual=0.0019334560178270043
   DEBUG:root:i=577 residual=0.001914802404537875
   DEBUG:root:i=578 residual=0.0018963287782933348
   DEBUG:root:i=579 residual=0.0018780334020131564
   DEBUG:root:i=580 residual=0.0018599145553766015
   DEBUG:root:i=581 residual=0.0018419705346750294
   DEBUG:root:i=582 residual=0.0018241996526758993
   DEBUG:root:i=583 residual=0.0018066002384040177
   DEBUG:root:i=584 residual=0.0017891706370635346
   DEBUG:root:i=585 residual=0.0017719092097748506
   DEBUG:root:i=586 residual=0.0017548143335515444
   DEBUG:root:i=587 residual=0.0017378844009980278
   DEBUG:root:i=588 residual=0.001721117820358854
   DEBUG:root:i=589 residual=0.0017045130150913647
   DEBUG:root:i=590 residual=0.0016880684240017482
   DEBUG:root:i=591 residual=0.001671782500935476
   DEBUG:root:i=592 residual=0.0016556537146113254
   DEBUG:root:i=593 residual=0.0016396805486083986
   DEBUG:root:i=594 residual=0.0016238615011041314
   DEBUG:root:i=595 residual=0.001608195084802274
   DEBUG:root:i=596 residual=0.0015926798267247037
   DEBUG:root:i=597 residual=0.0015773142681735236
   DEBUG:root:i=598 residual=0.0015620969644653357
   DEBUG:root:i=599 residual=0.0015470264848880435
   DEBUG:root:i=600 residual=0.0015321014126049412
   DEBUG:root:i=601 residual=0.001517320344365321
   DEBUG:root:i=602 residual=0.0015026818905143472
   DEBUG:root:i=603 residual=0.0014881846747910247
   DEBUG:root:i=604 residual=0.0014738273342625023
   DEBUG:root:i=605 residual=0.0014596085190957428
   DEBUG:root:i=606 residual=0.0014455268925264116
   DEBUG:root:i=607 residual=0.0014315811306850638
   DEBUG:root:i=608 residual=0.0014177699225027011
   DEBUG:root:i=609 residual=0.0014040919695073512
   DEBUG:root:i=610 residual=0.0013905459858323623
   DEBUG:root:i=611 residual=0.0013771306979886092
   DEBUG:root:i=612 residual=0.0013638448447560402
   DEBUG:root:i=613 residual=0.0013506871771613853
   DEBUG:root:i=614 residual=0.0013376564582274766
   DEBUG:root:i=615 residual=0.0013247514629075507
   DEBUG:root:i=616 residual=0.0013119709780394634
   DEBUG:root:i=617 residual=0.0012993138021345453
   DEBUG:root:i=618 residual=0.001286778745286984
   DEBUG:root:i=619 residual=0.0012743646291240817
   DEBUG:root:i=620 residual=0.0012620702865896252
   DEBUG:root:i=621 residual=0.0012498945619517127
   DEBUG:root:i=622 residual=0.0012378363105870552
   DEBUG:root:i=623 residual=0.0012258943989400086
   DEBUG:root:i=624 residual=0.0012140677044094803
   DEBUG:root:i=625 residual=0.0012023551151951958
   DEBUG:root:i=626 residual=0.0011907555302634624
   DEBUG:root:i=627 residual=0.0011792678591842555
   DEBUG:root:i=628 residual=0.0011678910220763485
   DEBUG:root:i=629 residual=0.0011566239494549143
   DEBUG:root:i=630 residual=0.0011454655821580284
   DEBUG:root:i=631 residual=0.0011344148712902863
   DEBUG:root:i=632 residual=0.0011234707780430524
   DEBUG:root:i=633 residual=0.001112632273624242
   DEBUG:root:i=634 residual=0.001101898339179863
   DEBUG:root:i=635 residual=0.0010912679657293387
   DEBUG:root:i=636 residual=0.0010807401539552788
   DEBUG:root:i=637 residual=0.0010703139142615873
   DEBUG:root:i=638 residual=0.0010599882665622653
   DEBUG:root:i=639 residual=0.0010497622402273314
   DEBUG:root:i=640 residual=0.0010396348740103645
   DEBUG:root:i=641 residual=0.0010296052159486989
   DEBUG:root:i=642 residual=0.0010196723232562695
   DEBUG:root:i=643 residual=0.0010098352622214426
   DEBUG:root:i=644 residual=0.0010000931082103197
   DEBUG:root:i=645 residual=0.0009904449454383962
   DEBUG:root:i=646 residual=0.0009808898669794338
   DEBUG:root:i=647 residual=0.0009714269746968353
   DEBUG:root:i=648 residual=0.000962055379107237
   DEBUG:root:i=649 residual=0.000952774199302235
   DEBUG:root:i=650 residual=0.0009435825628703815
   DEBUG:root:i=651 residual=0.0009344796057946348
   DEBUG:root:i=652 residual=0.000925464472471683
   DEBUG:root:i=653 residual=0.0009165363154834039
   DEBUG:root:i=654 residual=0.0009076942956111049
   DEBUG:root:i=655 residual=0.0008989375817770878
   DEBUG:root:i=656 residual=0.00089026535086495
   DEBUG:root:i=657 residual=0.0008816767877095089
   DEBUG:root:i=658 residual=0.0008731710850727185
   DEBUG:root:i=659 residual=0.0008647474434208099
   DEBUG:root:i=660 residual=0.0008564050709909051
   DEBUG:root:i=661 residual=0.0008481431836715788
   DEBUG:root:i=662 residual=0.0008399610048603137
   DEBUG:root:i=663 residual=0.0008318577654846423
   DEBUG:root:i=664 residual=0.0008238327039291486
   DEBUG:root:i=665 residual=0.000815885065840275
   DEBUG:root:i=666 residual=0.00080801410423091
   DEBUG:root:i=667 residual=0.0008002190792795725
   DEBUG:root:i=668 residual=0.0007924992583273083
   DEBUG:root:i=669 residual=0.0007848539157364938
   DEBUG:root:i=670 residual=0.0007772823329402214
   DEBUG:root:i=671 residual=0.0007697837982248701
   DEBUG:root:i=672 residual=0.0007623576068654491
   DEBUG:root:i=673 residual=0.0007550030607924614
   DEBUG:root:i=674 residual=0.0007477194687807445
   DEBUG:root:i=675 residual=0.0007405061462426939
   DEBUG:root:i=676 residual=0.0007333624151593788
   DEBUG:root:i=677 residual=0.0007262876041123625
   DEBUG:root:i=678 residual=0.0007192810481517598
   DEBUG:root:i=679 residual=0.0007123420887007844
   DEBUG:root:i=680 residual=0.0007054700735761559
   DEBUG:root:i=681 residual=0.0006986643569062531
   DEBUG:root:i=682 residual=0.0006919242990008367
   DEBUG:root:i=683 residual=0.0006852492663525778
   DEBUG:root:i=684 residual=0.0006786386316179287
   DEBUG:root:i=685 residual=0.000672091773439385
   DEBUG:root:i=686 residual=0.0006656080765040788
   DEBUG:root:i=687 residual=0.0006591869314292106
   DEBUG:root:i=688 residual=0.0006528277346752292
   DEBUG:root:i=689 residual=0.0006465298885665882
   DEBUG:root:i=690 residual=0.0006402928012107813
   DEBUG:root:i=691 residual=0.0006341158863434817
   DEBUG:root:i=692 residual=0.0006279985634432751
   DEBUG:root:i=693 residual=0.0006219402575750364
   DEBUG:root:i=694 residual=0.0006159403992821109
   DEBUG:root:i=695 residual=0.0006099984247078162
   DEBUG:root:i=696 residual=0.0006041137753787048
   DEBUG:root:i=697 residual=0.0005982858982061017
   DEBUG:root:i=698 residual=0.0005925142454195841
   DEBUG:root:i=699 residual=0.000586798274617247
   DEBUG:root:i=700 residual=0.0005811374485632189
   DEBUG:root:i=701 residual=0.000575531235184197
   DEBUG:root:i=702 residual=0.0005699791076108152
   DEBUG:root:i=703 residual=0.0005644805440036189
   DEBUG:root:i=704 residual=0.0005590350275985919
   DEBUG:root:i=705 residual=0.0005536420465737759
   DEBUG:root:i=706 residual=0.0005483010940934238
   DEBUG:root:i=707 residual=0.0005430116681605179
   DEBUG:root:i=708 residual=0.0005377732716665631
   DEBUG:root:i=709 residual=0.0005325854122769599
   DEBUG:root:i=710 residual=0.0005274476024281184
   DEBUG:root:i=711 residual=0.0005223593592175104
   DEBUG:root:i=712 residual=0.000517320204454884
   DEBUG:root:i=713 residual=0.0005123296645366398
   DEBUG:root:i=714 residual=0.0005073872704106503
   DEBUG:root:i=715 residual=0.000502492557606163
   DEBUG:root:i=716 residual=0.0004976450660712055
   DEBUG:root:i=717 residual=0.0004928443402527165
   DEBUG:root:i=718 residual=0.00048808992891805606
   DEBUG:root:i=719 residual=0.0004833813852554238
   DEBUG:root:i=720 residual=0.00047871826675881154
   DEBUG:root:i=721 residual=0.00047410013514282296
   DEBUG:root:i=722 residual=0.0004695265564180473
   DEBUG:root:i=723 residual=0.0004649971007226965
   DEBUG:root:i=724 residual=0.000460511342340434
   DEBUG:root:i=725 residual=0.0004560688597356954
   DEBUG:root:i=726 residual=0.000451669235391679
   DEBUG:root:i=727 residual=0.0004473120557781743
   DEBUG:root:i=728 residual=0.0004429969114463891
   DEBUG:root:i=729 residual=0.00043872339681295963
   DEBUG:root:i=730 residual=0.00043449111027868317
   DEBUG:root:i=731 residual=0.00043029965407563765
   DEBUG:root:i=732 residual=0.00042614863426499664
   DEBUG:root:i=733 residual=0.00042203766072957825
   DEBUG:root:i=734 residual=0.00041796634714216823
   DEBUG:root:i=735 residual=0.00041393431086175384
   DEBUG:root:i=736 residual=0.00040994117293599765
   DEBUG:root:i=737 residual=0.0004059865581079395
   DEBUG:root:i=738 residual=0.0004020700947033281
   DEBUG:root:i=739 residual=0.00039819141465600197
   DEBUG:root:i=740 residual=0.00039435015344897067
   DEBUG:root:i=741 residual=0.00039054595008655344
   DEBUG:root:i=742 residual=0.00038677844701563946
   DEBUG:root:i=743 residual=0.0003830472901955594
   DEBUG:root:i=744 residual=0.0003793521289507859
   DEBUG:root:i=745 residual=0.0003756926160268759
   DEBUG:root:i=746 residual=0.000372068407476724
   DEBUG:root:i=747 residual=0.00036847916273642395
   DEBUG:root:i=748 residual=0.00036492454445258727
   DEBUG:root:i=749 residual=0.0003614042185831547
   DEBUG:root:i=750 residual=0.0003579178542735158
   DEBUG:root:i=751 residual=0.0003544651239080011
   DEBUG:root:i=752 residual=0.0003510457029721458
   DEBUG:root:i=753 residual=0.0003476592700984827
   DEBUG:root:i=754 residual=0.0003443055070564341
   DEBUG:root:i=755 residual=0.0003409840986519214
   DEBUG:root:i=756 residual=0.00033769473276458705
   DEBUG:root:i=757 residual=0.0003344371002201579
   DEBUG:root:i=758 residual=0.0003312108949000153
   DEBUG:root:i=759 residual=0.0003280158136080632
   DEBUG:root:i=760 residual=0.00032485155607414937
   DEBUG:root:i=761 residual=0.00032171782489804365
   DEBUG:root:i=762 residual=0.00031861432559756805
   DEBUG:root:i=763 residual=0.0003155407665276979
   DEBUG:root:i=764 residual=0.0003124968588020525
   DEBUG:root:i=765 residual=0.00030948231638794674
   DEBUG:root:i=766 residual=0.00030649685597363297
   DEBUG:root:i=767 residual=0.0003035401969956019
   DEBUG:root:i=768 residual=0.00030061206159339565
   DEBUG:root:i=769 residual=0.00029771217455436846
   DEBUG:root:i=770 residual=0.00029484026339954094
   DEBUG:root:i=771 residual=0.00029199605820716347
   DEBUG:root:i=772 residual=0.0002891792916884298
   DEBUG:root:i=773 residual=0.0002863896991331444
   DEBUG:root:i=774 residual=0.00028362701838823256
   DEBUG:root:i=775 residual=0.00028089098979287554
   DEBUG:root:i=776 residual=0.0002781813562698187
   DEBUG:root:i=777 residual=0.0002754978631338628
   DEBUG:root:i=778 residual=0.0002728402582217315
   DEBUG:root:i=779 residual=0.0002702082917545295
   DEBUG:root:i=780 residual=0.0002676017163996062
   DEBUG:root:i=781 residual=0.00026502028721003395
   DEBUG:root:i=782 residual=0.000262463761580539
   DEBUG:root:i=783 residual=0.0002599318992547205
   DEBUG:root:i=784 residual=0.0002574244622856814
   DEBUG:root:i=785 residual=0.0002549412150631257
   DEBUG:root:i=786 residual=0.00025248192418045654
   DEBUG:root:i=787 residual=0.0002500463585513749
   DEBUG:root:i=788 residual=0.0002476342892659985
   DEBUG:root:i=789 residual=0.000245245489663401
   DEBUG:root:i=790 residual=0.00024287973524242943
   DEBUG:root:i=791 residual=0.00024053680368956977
   DEBUG:root:i=792 residual=0.00023821647481467672
   DEBUG:root:i=793 residual=0.0002359185305490574
   DEBUG:root:i=794 residual=0.0002336427549691223
   DEBUG:root:i=795 residual=0.00023138893417361164
   DEBUG:root:i=796 residual=0.00022915685639809025
   DEBUG:root:i=797 residual=0.0002269463118203483
   DEBUG:root:i=798 residual=0.0002247570927581443
   DEBUG:root:i=799 residual=0.00022258899344390048
   DEBUG:root:i=800 residual=0.00022044181015521628
   DEBUG:root:i=801 residual=0.0002183153410918447
   DEBUG:root:i=802 residual=0.00021620938642345465
   DEBUG:root:i=803 residual=0.00021412374821515664
   DEBUG:root:i=804 residual=0.00021205823049003477
   DEBUG:root:i=805 residual=0.0002100126391397048
   DEBUG:root:i=806 residual=0.00020798678193850633
   DEBUG:root:i=807 residual=0.00020598046846423838
   DEBUG:root:i=808 residual=0.00020399351021827355
   DEBUG:root:i=809 residual=0.00020202572045048973
   DEBUG:root:i=810 residual=0.0002000769142444203
   DEBUG:root:i=811 residual=0.00019814690846300996
   DEBUG:root:i=812 residual=0.00019623552171449434
   DEBUG:root:i=813 residual=0.00019434257439829132
   DEBUG:root:i=814 residual=0.00019246788860339043
   DEBUG:root:i=815 residual=0.0001906112881841541
   DEBUG:root:i=816 residual=0.00018877259863289293
   DEBUG:root:i=817 residual=0.00018695164718171843
   DEBUG:root:i=818 residual=0.0001851482626839458
   DEBUG:root:i=819 residual=0.00018336227570579067
   DEBUG:root:i=820 residual=0.00018159351836976556
   DEBUG:root:i=821 residual=0.00017984182445620342
   DEBUG:root:i=822 residual=0.0001781070293919041
   DEBUG:root:i=823 residual=0.00017638897009883642
   DEBUG:root:i=824 residual=0.00017468748516293872
   DEBUG:root:i=825 residual=0.00017300241465874344
   DEBUG:root:i=826 residual=0.00017133360022565512
   DEBUG:root:i=827 residual=0.0001696808850921662
   DEBUG:root:i=828 residual=0.00016804411387908732
   DEBUG:root:i=829 residual=0.0001664231328041718
   DEBUG:root:i=830 residual=0.00016481778953448452
   DEBUG:root:i=831 residual=0.00016322793320566728
   DEBUG:root:i=832 residual=0.00016165341441012483
   DEBUG:root:i=833 residual=0.0001600940851591113
   DEBUG:root:i=834 residual=0.00015854979897217503
   DEBUG:root:i=835 residual=0.00015702041067867066
   DEBUG:root:i=836 residual=0.0001555057765712594
   DEBUG:root:i=837 residual=0.0001540057543135921
   DEBUG:root:i=838 residual=0.00015252020294363093
   DEBUG:root:i=839 residual=0.00015104898284881246
   DEBUG:root:i=840 residual=0.00014959195575097822
   DEBUG:root:i=841 residual=0.00014814898480688878
   DEBUG:root:i=842 residual=0.00014671993432101407
   DEBUG:root:i=843 residual=0.0001453046700610744
   DEBUG:root:i=844 residual=0.0001439030589843466
   DEBUG:root:i=845 residual=0.00014251496939759315
   DEBUG:root:i=846 residual=0.00014114027088353036
   DEBUG:root:i=847 residual=0.00013977883422212995
   DEBUG:root:i=848 residual=0.00013843053147238635
   DEBUG:root:i=849 residual=0.00013709523592785145
   DEBUG:root:i=850 residual=0.00013577282212088316
   DEBUG:root:i=851 residual=0.00013446316577823834
   DEBUG:root:i=852 residual=0.00013316614381004562
   DEBUG:root:i=853 residual=0.00013188163435360833
   DEBUG:root:i=854 residual=0.00013060951667481148
   DEBUG:root:i=855 residual=0.0001293496712411308
   DEBUG:root:i=856 residual=0.0001281019796708918
   DEBUG:root:i=857 residual=0.00012686632468079053
   DEBUG:root:i=858 residual=0.0001256425901764675
   DEBUG:root:i=859 residual=0.00012443066116462703
   DEBUG:root:i=860 residual=0.00012323042372801556
   DEBUG:root:i=861 residual=0.0001220417650976399
   DEBUG:root:i=862 residual=0.0001208645735793062
   DEBUG:root:i=863 residual=0.00011969873853345927
   DEBUG:root:i=864 residual=0.00011854415037931484
   DEBUG:root:i=865 residual=0.00011740070066235863
   DEBUG:root:i=866 residual=0.00011626828189811204
   DEBUG:root:i=867 residual=0.00011514678769288628
   DEBUG:root:i=868 residual=0.00011403611262714951
   DEBUG:root:i=869 residual=0.0001129361523312837
   DEBUG:root:i=870 residual=0.00011184680346239804
   DEBUG:root:i=871 residual=0.00011076796362931184
   DEBUG:root:i=872 residual=0.00010969953144646226
   DEBUG:root:i=873 residual=0.00010864140655202649
   DEBUG:root:i=874 residual=0.00010759348944747986
   DEBUG:root:i=875 residual=0.00010655568169606834
   DEBUG:root:i=876 residual=0.00010552788576134019
   DEBUG:root:i=877 residual=0.00010451000505139298
   DEBUG:root:i=878 residual=0.00010350194393192807
   DEBUG:root:i=879 residual=0.00010250360765295323
   DEBUG:root:i=880 residual=0.00010151490239734724
   DEBUG:root:i=881 residual=0.00010053573527504755
   DEBUG:root:i=882 residual=9.956601424939477e-05
   DEBUG:root:i=883 residual=9.860564820494589e-05
   DEBUG:root:i=884 residual=9.765454686436595e-05
   DEBUG:root:i=885 residual=9.671262090420852e-05
   DEBUG:root:i=886 residual=9.577978177482946e-05
   DEBUG:root:i=887 residual=9.485594180198641e-05
   DEBUG:root:i=888 residual=9.394101419082347e-05
   DEBUG:root:i=889 residual=9.303491295273464e-05
   DEBUG:root:i=890 residual=9.213755295338747e-05
   DEBUG:root:i=891 residual=9.124884984440312e-05
   DEBUG:root:i=892 residual=9.036872013514149e-05
   DEBUG:root:i=893 residual=8.949708108106227e-05
   DEBUG:root:i=894 residual=8.863385081903449e-05
   DEBUG:root:i=895 residual=8.77789481774152e-05
   DEBUG:root:i=896 residual=8.69322928794362e-05
   DEBUG:root:i=897 residual=8.609380529023641e-05
   DEBUG:root:i=898 residual=8.526340667901849e-05
   DEBUG:root:i=899 residual=8.444101897512567e-05
   DEBUG:root:i=900 residual=8.362656491819973e-05
   DEBUG:root:i=901 residual=8.281996794707194e-05
   DEBUG:root:i=902 residual=8.202115227020288e-05
   DEBUG:root:i=903 residual=8.12300428380154e-05
   DEBUG:root:i=904 residual=8.044656530269792e-05
   DEBUG:root:i=905 residual=7.967064601235495e-05
   DEBUG:root:i=906 residual=7.890221208498511e-05
   DEBUG:root:i=907 residual=7.814119130430605e-05
   DEBUG:root:i=908 residual=7.738751211566439e-05
   DEBUG:root:i=909 residual=7.664110374654278e-05
   DEBUG:root:i=910 residual=7.590189602653333e-05
   DEBUG:root:i=911 residual=7.516981950352442e-05
   DEBUG:root:i=912 residual=7.444480536542314e-05
   DEBUG:root:i=913 residual=7.372678548838663e-05
   DEBUG:root:i=914 residual=7.301569240268168e-05
   DEBUG:root:i=915 residual=7.231145929648307e-05
   DEBUG:root:i=916 residual=7.161401996297257e-05
   DEBUG:root:i=917 residual=7.092330889607254e-05
   DEBUG:root:i=918 residual=7.023926114030777e-05
   DEBUG:root:i=919 residual=6.95618124647375e-05
   DEBUG:root:i=920 residual=6.889089918817937e-05
   DEBUG:root:i=921 residual=6.822645824141604e-05
   DEBUG:root:i=922 residual=6.756842721580558e-05
   DEBUG:root:i=923 residual=6.691674425619557e-05
   DEBUG:root:i=924 residual=6.62713481180999e-05
   DEBUG:root:i=925 residual=6.56321781876137e-05
   DEBUG:root:i=926 residual=6.499917435893155e-05
   DEBUG:root:i=927 residual=6.437227717062072e-05
   DEBUG:root:i=928 residual=6.375142771741812e-05
   DEBUG:root:i=929 residual=6.313656761579131e-05
   DEBUG:root:i=930 residual=6.252763913020078e-05
   DEBUG:root:i=931 residual=6.19245850173859e-05
   DEBUG:root:i=932 residual=6.132734862256409e-05
   DEBUG:root:i=933 residual=6.0735873802102696e-05
   DEBUG:root:i=934 residual=6.015010498089874e-05
   DEBUG:root:i=935 residual=5.956998712327209e-05
   DEBUG:root:i=936 residual=5.8995465703375494e-05
   DEBUG:root:i=937 residual=5.842648672922229e-05
   DEBUG:root:i=938 residual=5.786299675172584e-05
   DEBUG:root:i=939 residual=5.730494279224254e-05
   DEBUG:root:i=940 residual=5.6752272424006166e-05
   DEBUG:root:i=941 residual=5.620493370432042e-05
   DEBUG:root:i=942 residual=5.5662875203114635e-05
   DEBUG:root:i=943 residual=5.5126045993459374e-05
   DEBUG:root:i=944 residual=5.459439561659431e-05
   DEBUG:root:i=945 residual=5.406787409631235e-05
   DEBUG:root:i=946 residual=5.354643197784682e-05
   DEBUG:root:i=947 residual=5.303002026468665e-05
   DEBUG:root:i=948 residual=5.251859040244013e-05
   DEBUG:root:i=949 residual=5.2012094347957774e-05
   DEBUG:root:i=950 residual=5.151048450537301e-05
   DEBUG:root:i=951 residual=5.101371373524361e-05
   DEBUG:root:i=952 residual=5.0521735349891865e-05
   DEBUG:root:i=953 residual=5.003450311260673e-05
   DEBUG:root:i=954 residual=4.955197125733637e-05
   DEBUG:root:i=955 residual=4.9074094415179876e-05
   DEBUG:root:i=956 residual=4.8600827692064195e-05
   DEBUG:root:i=957 residual=4.813212663427988e-05
   DEBUG:root:i=958 residual=4.7667947160499496e-05
   DEBUG:root:i=959 residual=4.720824566301778e-05
   DEBUG:root:i=960 residual=4.675297893504039e-05
   DEBUG:root:i=961 residual=4.6302104233666117e-05
   DEBUG:root:i=962 residual=4.58555791431685e-05
   DEBUG:root:i=963 residual=4.541336173039104e-05
   DEBUG:root:i=964 residual=4.497541041372475e-05
   DEBUG:root:i=965 residual=4.4541684078744666e-05
   DEBUG:root:i=966 residual=4.4112141923349673e-05
   DEBUG:root:i=967 residual=4.3686743602211654e-05
   DEBUG:root:i=968 residual=4.326544913493188e-05
   DEBUG:root:i=969 residual=4.284821894000874e-05
   DEBUG:root:i=970 residual=4.243501382451856e-05
   DEBUG:root:i=971 residual=4.202579491064759e-05
   DEBUG:root:i=972 residual=4.1620523781872155e-05
   DEBUG:root:i=973 residual=4.121916234639259e-05
   DEBUG:root:i=974 residual=4.08216728905181e-05
   DEBUG:root:i=975 residual=4.0428018034401256e-05
   DEBUG:root:i=976 residual=4.0038160848712305e-05
   DEBUG:root:i=977 residual=3.9652064611301795e-05
   DEBUG:root:i=978 residual=3.926969311812167e-05
   DEBUG:root:i=979 residual=3.889101038628679e-05
   DEBUG:root:i=980 residual=3.851598085847825e-05
   DEBUG:root:i=981 residual=3.814456926132531e-05
   DEBUG:root:i=982 residual=3.777674073192779e-05
   DEBUG:root:i=983 residual=3.7412460680918014e-05
   DEBUG:root:i=984 residual=3.7051694875676505e-05
   DEBUG:root:i=985 residual=3.669440943496557e-05
   DEBUG:root:i=986 residual=3.6340570765364694e-05
   DEBUG:root:i=987 residual=3.599014561950536e-05
   DEBUG:root:i=988 residual=3.564310106189905e-05
   DEBUG:root:i=989 residual=3.529940448835275e-05
   DEBUG:root:i=990 residual=3.495902360572707e-05
   DEBUG:root:i=991 residual=3.46219264132259e-05
   DEBUG:root:i=992 residual=3.428808124008323e-05
   DEBUG:root:i=993 residual=3.395745670753134e-05
   DEBUG:root:i=994 residual=3.363002176153153e-05
   DEBUG:root:i=995 residual=3.330574563933733e-05
   DEBUG:root:i=996 residual=3.298459782575657e-05
   DEBUG:root:i=997 residual=3.266654818325168e-05
   DEBUG:root:i=998 residual=3.235156680671249e-05
   DEBUG:root:i=999 residual=3.203962409621269e-05
   INFO:root:rank=0 pagerank=5.2387e+01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
   INFO:root:rank=1 pagerank=5.2387e+01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
   INFO:root:rank=2 pagerank=7.9442e+00 url=www.lawfareblog.com/cost-using-zero-days
   INFO:root:rank=3 pagerank=2.3701e+00 url=www.lawfareblog.com/lawfare-podcast-former-congressman-brian-baird-and-daniel-schuman-how-congress-can-continue-function
   INFO:root:rank=4 pagerank=1.5534e+00 url=www.lawfareblog.com/events
   INFO:root:rank=5 pagerank=1.1876e+00 url=www.lawfareblog.com/water-wars-increased-us-focus-indo-pacific
   INFO:root:rank=6 pagerank=1.1876e+00 url=www.lawfareblog.com/water-wars-drill-maybe-drill
   INFO:root:rank=7 pagerank=1.1876e+00 url=www.lawfareblog.com/water-wars-disjointed-operations-south-china-sea
   INFO:root:rank=8 pagerank=1.1876e+00 url=www.lawfareblog.com/water-wars-song-oil-and-fire
   INFO:root:rank=9 pagerank=1.1876e+00 url=www.lawfareblog.com/water-wars-sinking-feeling-philippine-china-relations
   ```

   Task 2, part 1:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona'
   INFO:root:rank=0 pagerank=8.8870e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
   INFO:root:rank=1 pagerank=8.8867e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
   INFO:root:rank=2 pagerank=1.8256e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
   INFO:root:rank=3 pagerank=1.4907e-01 url=www.lawfareblog.com/rational-security-my-corona-edition
   INFO:root:rank=4 pagerank=1.4907e-01 url=www.lawfareblog.com/brexit-not-immune-coronavirus
   INFO:root:rank=5 pagerank=1.0729e-01 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
   INFO:root:rank=6 pagerank=1.0199e-01 url=www.lawfareblog.com/britains-coronavirus-response
   INFO:root:rank=7 pagerank=1.0199e-01 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
   INFO:root:rank=8 pagerank=9.4298e-02 url=www.lawfareblog.com/lawfare-podcast-mom-and-dad-talk-clinical-trials-pandemic
   INFO:root:rank=9 pagerank=8.7207e-02 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
   ```

   Task 2, part 2:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona' --search_query='-corona'
   INFO:root:rank=0 pagerank=8.8870e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
   INFO:root:rank=1 pagerank=8.8867e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
   INFO:root:rank=2 pagerank=1.8256e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
   INFO:root:rank=3 pagerank=1.0729e-01 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
   INFO:root:rank=4 pagerank=9.4298e-02 url=www.lawfareblog.com/lawfare-podcast-mom-and-dad-talk-clinical-trials-pandemic
   INFO:root:rank=5 pagerank=7.9633e-02 url=www.lawfareblog.com/fault-lines-foreign-policy-quarantined
   INFO:root:rank=6 pagerank=7.5307e-02 url=www.lawfareblog.com/limits-world-health-organization
   INFO:root:rank=7 pagerank=6.8115e-02 url=www.lawfareblog.com/chinatalk-dispatches-shanghai-beijing-and-hong-kong
   INFO:root:rank=8 pagerank=6.4847e-02 url=www.lawfareblog.com/us-moves-dismiss-case-against-company-linked-ira-troll-farm
   INFO:root:rank=9 pagerank=6.4847e-02 url=www.lawfareblog.com/livestream-house-armed-services-committee-holds-hearing-priorities-missile-defense
   ```

1. Ensure that all your changes to the `pagerank.py` and `README.md` files are committed to your repo and pushed to github.

1. Get at least 5 stars on your repo.
   (You made trade stars with other students in the class.)

   > **NOTE:**
   > 
   > Recruiters use github profiles to determine who to hire,
   > and pagerank is used to rank user profiles and projects.
   > Links in this graph correspond to who has starred/followed who's repo.
   > By getting more stars on your repo, you'll be increasing your github pagerank, which increases the likelihood that recruiters will hire you.
   > To see an example, [perform a search for `data mining`](https://github.com/search?q=data+mining).
   > Notice that the results are returned "approximately" ranked by the number of stars,
   > but because "some stars count more than others" the results are not exactly ranked by the number of stars.
   > (I asked you not to fork this repo because forks are ranked lower than non-forks.)
   >
   > In some sense, we are doing a "dual problem" to data mining by getting these stars.
   > Recruiters are using data mining to find out who the best people to recruit are,
   > and we are hacking their data mining algorithms by making those algorithms select you instead of someone else.
   >
   > If you're interested in exploring this idea further, here's a python tutorial for extracting GitHub's social graph: <https://www.oreilly.com/library/view/mining-the-social/9781449368180/ch07.html> ; if you're interested in learning more about how recruiters use github profiles, read this Hacker News post: <https://news.ycombinator.com/item?id=19413348>.

1. Submit the url of your repo to sakai.

   Each part is worth 2 points, for 12 points overall.
