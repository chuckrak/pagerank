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
INFO:root:rank=0 pagerank=4.5861e-03 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=1 pagerank=4.0460e-03 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
INFO:root:rank=2 pagerank=2.6116e-03 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=3 pagerank=2.5390e-03 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=4 pagerank=2.3557e-03 url=www.lawfareblog.com/israeli-emergency-regulations-location-tracking-coronavirus-carriers
INFO:root:rank=5 pagerank=2.2895e-03 url=www.lawfareblog.com/why-congress-conducting-business-usual-face-coronavirus
INFO:root:rank=6 pagerank=2.2727e-03 url=www.lawfareblog.com/livestream-house-oversight-committee-holds-hearing-government-coronavirus-response
INFO:root:rank=7 pagerank=2.2520e-03 url=www.lawfareblog.com/congressional-homeland-security-committees-seek-ways-support-state-federal-responses-coronavirus
INFO:root:rank=8 pagerank=2.1878e-03 url=www.lawfareblog.com/paper-hearing-experts-debate-digital-contact-tracing-and-coronavirus-privacy-concerns
INFO:root:rank=9 pagerank=2.0339e-03 url=www.lawfareblog.com/cyberlaw-podcast-how-israel-fighting-coronavirus

$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='trump'
INFO:root:rank=0 pagerank=7.5497e-08 url=www.lawfareblog.com/trumps-doj-assert-state-secrets-privilege-salim-v-mitchell
INFO:root:rank=1 pagerank=7.3255e-08 url=www.lawfareblog.com/i-hope-instance-fake-news-fbi-messages-show-bureaus-real-reaction-trump-firing-james-comey
INFO:root:rank=2 pagerank=6.6809e-08 url=www.lawfareblog.com/eu-turkey-deal-expediency-trumps-eus-higher-calling
INFO:root:rank=3 pagerank=6.5992e-08 url=www.lawfareblog.com/how-will-we-know-if-russia-trump-investigations-congress-and-fbi-are-credible
INFO:root:rank=4 pagerank=6.5838e-08 url=www.lawfareblog.com/international-law-age-trump-post-human-rights-agenda
INFO:root:rank=5 pagerank=6.5620e-08 url=www.lawfareblog.com/judge-enjoins-trump-administrations-easing-restrictions-3-d-gun-blueprints
INFO:root:rank=6 pagerank=6.2893e-08 url=www.lawfareblog.com/trump-wants-bigger-better-deal-iran-what-does-tehran-want
INFO:root:rank=7 pagerank=5.6700e-13 url=www.lawfareblog.com/trumps-misplaced-reliance-justice-department-memoranda-protect-his-financial-records
INFO:root:rank=8 pagerank=0.0000e+00 url=www.lawfareblog.com/donald-trump-and-politically-weaponized-executive-branch
INFO:root:rank=9 pagerank=0.0000e+00 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns


$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='iran'
INFO:root:rank=0 pagerank=4.5430e-08 url=www.lawfareblog.com/adjusting-operations-libya-suit-wpr-clock
INFO:root:rank=1 pagerank=2.9199e-08 url=www.lawfareblog.com/best-served-cold-responding-iranian-protests
INFO:root:rank=2 pagerank=2.4164e-08 url=www.lawfareblog.com/cyber-reform-israel-impasse-primer
INFO:root:rank=3 pagerank=2.3825e-08 url=www.lawfareblog.com/dignity-and-needs-young-syrian-refugees-middle-east
INFO:root:rank=4 pagerank=2.3680e-08 url=www.lawfareblog.com/early-report-indicates-ubl-killed-drone-strike-pakistanhow-will-drone-critics-respond
INFO:root:rank=5 pagerank=2.3395e-08 url=www.lawfareblog.com/house-divided-why-partitioning-libya-might-be-only-way-save-it
INFO:root:rank=6 pagerank=2.3316e-08 url=www.lawfareblog.com/implications-eu-delisting-bank-saderat-iran
INFO:root:rank=7 pagerank=2.2361e-08 url=www.lawfareblog.com/two-small-steps-long-road-suppressing-unlawful-security-interrogations-israel
INFO:root:rank=8 pagerank=2.2361e-08 url=www.lawfareblog.com/panetta-sessions-exchange-authorization-force-syria
INFO:root:rank=9 pagerank=2.2361e-08 url=www.lawfareblog.com/relatively-weak-article-ii-basis-bombing-iraq-and-syria-and-remember-presidents-august-31-2013
```

**Part 3:**

The webgraph of lawfareblog.com (i.e. the `P` matrix) naturally contains a lot of structure.
For example, essentially all pages on the domain have links to the root page <https://lawfareblog.com/> and other "non-article" pages like <https://www.lawfareblog.com/topics> and <https://www.lawfareblog.com/subscribe-lawfare>.
These pages therefore have a large pagerank.
We can get a list of the pages with the largest pagerank by running

```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz
INFO:root:rank=0 pagerank=8.4167e+00 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=1 pagerank=8.4167e+00 url=www.lawfareblog.com/snowden-revelations
INFO:root:rank=2 pagerank=8.4167e+00 url=www.lawfareblog.com/support-lawfare
INFO:root:rank=3 pagerank=8.4167e+00 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=4 pagerank=8.4167e+00 url=www.lawfareblog.com/our-comments-policy
INFO:root:rank=5 pagerank=8.4167e+00 url=www.lawfareblog.com/subscribe-lawfare
INFO:root:rank=6 pagerank=8.4167e+00 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban
INFO:root:rank=7 pagerank=8.4167e+00 url=www.lawfareblog.com/cyberlaw-podcast-sandworm-and-grus-global-intifada
INFO:root:rank=8 pagerank=8.4167e+00 url=www.lawfareblog.com/cyberlaw-podcast-plumbing-depths-artificial-stupidity
INFO:root:rank=9 pagerank=8.4167e+00 url=www.lawfareblog.com/doj-charges-two-former-twitter-employees-spying-saudis
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
INFO:root:rank=0 pagerank=4.6091e+00 url=www.lawfareblog.com/0-days-n-days-iphones-and-android
INFO:root:rank=1 pagerank=2.9867e+00 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=2 pagerank=2.9669e+00 url=www.lawfareblog.com/doj-charges-two-former-twitter-employees-spying-saudis
INFO:root:rank=3 pagerank=2.0173e+00 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban
INFO:root:rank=4 pagerank=1.8769e+00 url=www.lawfareblog.com/subscribe-lawfare
INFO:root:rank=5 pagerank=1.8762e+00 url=www.lawfareblog.com/lessons-so-far-whatsapp-v-nso
INFO:root:rank=6 pagerank=1.8693e+00 url=www.lawfareblog.com/cyberlaw-podcast-plumbing-depths-artificial-stupidity
INFO:root:rank=7 pagerank=1.7655e+00 url=www.lawfareblog.com/our-comments-policy
INFO:root:rank=8 pagerank=1.6807e+00 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=9 pagerank=9.8345e-01 url=www.lawfareblog.com/cyberlaw-podcast-sandworm-and-grus-global-intifada
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
INFO:root:rank=0 pagerank=8.8870e-01 url=www.lawfareblog.com/0-days-n-days-iphones-and-android
INFO:root:rank=1 pagerank=8.8867e-01 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=2 pagerank=1.8256e-01 url=www.lawfareblog.com/doj-charges-two-former-twitter-employees-spying-saudis
INFO:root:rank=3 pagerank=1.4907e-01 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban
INFO:root:rank=4 pagerank=1.4907e-01 url=www.lawfareblog.com/subscribe-lawfare
INFO:root:rank=5 pagerank=1.0729e-01 url=www.lawfareblog.com/lessons-so-far-whatsapp-v-nso
INFO:root:rank=6 pagerank=1.0199e-01 url=www.lawfareblog.com/cyberlaw-podcast-plumbing-depths-artificial-stupidity
INFO:root:rank=7 pagerank=1.0199e-01 url=www.lawfareblog.com/our-comments-policy
INFO:root:rank=8 pagerank=9.4298e-02 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=9 pagerank=8.7207e-02 url=www.lawfareblog.com/cyberlaw-podcast-sandworm-and-grus-global-intifada
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
   INFO:gensim.models.keyedvectors:loading projection weights from /home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz
   DEBUG:smart_open.smart_open_lib:{'transport_params': None, 'compression': 'infer_from_extension', 'opener': None, 'closefd': True, 'newline': None, 'errors': None, 'encoding': None, 'buffering': -1, 'mode': 'rb', 'uri': '/home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz'}
   DEBUG:gensim.utils:starting a new internal lifecycle event log for KeyedVectors
   INFO:gensim.utils:KeyedVectors lifecycle event {'msg': 'loaded (3000000, 300) matrix of type float32 from /home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz', 'binary': True, 'encoding': 'utf8', 'datetime': '2022-12-11T20:34:55.970181', 'gensim': '4.2.0', 'python': '3.6.9 (default, Dec  8 2021, 21:08:43) \n[GCC 8.4.0]', 'platform': 'Linux-4.15.0-88-generic-x86_64-with-Ubuntu-18.04-bionic', 'event': 'load_word2vec_format'}
   DEBUG:root:computing indices
   DEBUG:root:computing values
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
   INFO:root:rank=0 pagerank=2.1634e+00 url=1
   INFO:root:rank=1 pagerank=1.6664e+00 url=2
   INFO:root:rank=2 pagerank=1.2402e+00 url=3
   INFO:root:rank=3 pagerank=4.5712e-01 url=5
   INFO:root:rank=4 pagerank=3.5619e-01 url=4
   INFO:root:rank=5 pagerank=3.2078e-01 url=6

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
   INFO:gensim.models.keyedvectors:loading projection weights from /home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz
   INFO:gensim.utils:KeyedVectors lifecycle event {'msg': 'loaded (3000000, 300) matrix of type float32 from /home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz', 'binary': True, 'encoding': 'utf8', 'datetime': '2022-12-11T20:03:42.012914', 'gensim': '4.2.0', 'python': '3.6.9 (default, Dec  8 2021, 21:08:43) \n[GCC 8.4.0]', 'platform': 'Linux-4.15.0-88-generic-x86_64-with-Ubuntu-18.04-bionic', 'event': 'load_word2vec_format'}
   INFO:root:rank=0 pagerank=7.5497e-08 url=www.lawfareblog.com/trumps-doj-assert-state-secrets-privilege-salim-v-mitchell
   INFO:root:rank=1 pagerank=7.3255e-08 url=www.lawfareblog.com/i-hope-instance-fake-news-fbi-messages-show-bureaus-real-reaction-trump-firing-james-comey
   INFO:root:rank=2 pagerank=6.6809e-08 url=www.lawfareblog.com/eu-turkey-deal-expediency-trumps-eus-higher-calling
   INFO:root:rank=3 pagerank=6.5992e-08 url=www.lawfareblog.com/how-will-we-know-if-russia-trump-investigations-congress-and-fbi-are-credible
   INFO:root:rank=4 pagerank=6.5838e-08 url=www.lawfareblog.com/international-law-age-trump-post-human-rights-agenda
   INFO:root:rank=5 pagerank=6.5620e-08 url=www.lawfareblog.com/judge-enjoins-trump-administrations-easing-restrictions-3-d-gun-blueprints
   INFO:root:rank=6 pagerank=6.2893e-08 url=www.lawfareblog.com/trump-wants-bigger-better-deal-iran-what-does-tehran-want
   INFO:root:rank=7 pagerank=5.6700e-13 url=www.lawfareblog.com/trumps-misplaced-reliance-justice-department-memoranda-protect-his-financial-records
   INFO:root:rank=8 pagerank=0.0000e+00 url=www.lawfareblog.com/donald-trump-and-politically-weaponized-executive-branch
   INFO:root:rank=9 pagerank=0.0000e+00 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns



   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='iran'
   INFO:gensim.models.keyedvectors:loading projection weights from /home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz
   INFO:gensim.utils:KeyedVectors lifecycle event {'msg': 'loaded (3000000, 300) matrix of type float32 from /home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz', 'binary': True, 'encoding': 'utf8', 'datetime': '2022-12-08T09:26:45.453343', 'gensim': '4.2.0', 'python': '3.6.9 (default, Dec  8 2021, 21:08:43) \n[GCC 8.4.0]', 'platform': 'Linux-4.15.0-88-generic-x86_64-with-Ubuntu-18.04-bionic', 'event': 'load_word2vec_format'}
   INFO:root:rank=0 pagerank=6.6135e-02 url=www.lawfareblog.com/praise-presidents-iran-tweets
   INFO:root:rank=1 pagerank=2.9199e-02 url=www.lawfareblog.com/how-us-iran-tensions-could-disrupt-iraqs-fragile-peace
   INFO:root:rank=2 pagerank=1.7709e-02 url=www.lawfareblog.com/cyber-command-operational-update-clarifying-june-2019-iran-operation
   INFO:root:rank=3 pagerank=1.4604e-02 url=www.lawfareblog.com/aborted-iran-strike-fine-line-between-necessity-and-revenge
   INFO:root:rank=4 pagerank=1.4578e-02 url=www.lawfareblog.com/its-not-only-iraq-and-syria
   INFO:root:rank=5 pagerank=1.1964e-02 url=www.lawfareblog.com/congress-us-policy-toward-syria-and-turkey-overview-recent-hearings
   INFO:root:rank=6 pagerank=1.0043e-02 url=www.lawfareblog.com/sixteen-year-old-american-islamic-state-fighter-reportedly-captured-syria
   INFO:root:rank=7 pagerank=8.4943e-03 url=www.lawfareblog.com/readings-post-syria-extracted-onion
   INFO:root:rank=8 pagerank=8.4512e-03 url=www.lawfareblog.com/iranian-hostage-crisis-and-its-effect-american-politics
   INFO:root:rank=9 pagerank=8.3989e-03 url=www.lawfareblog.com/parsing-state-departments-letter-use-force-against-irans


   ```

   Task 1, part 3:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz
   INFO:gensim.models.keyedvectors:loading projection weights from /home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz
   INFO:gensim.utils:KeyedVectors lifecycle event {'msg': 'loaded (3000000, 300) matrix of type float32 from /home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz', 'binary': True, 'encoding': 'utf8', 'datetime': '2022-12-11T20:38:10.929578', 'gensim': '4.2.0', 'python': '3.6.9 (default, Dec  8 2021, 21:08:43) \n[GCC 8.4.0]', 'platform': 'Linux-4.15.0-88-generic-x86_64-with-Ubuntu-18.04-bionic', 'event': 'load_word2vec_format'}
   INFO:root:rank=0 pagerank=8.4167e+00 url=www.lawfareblog.com/lawfare-job-board
   INFO:root:rank=1 pagerank=8.4167e+00 url=www.lawfareblog.com/snowden-revelations
   INFO:root:rank=2 pagerank=8.4167e+00 url=www.lawfareblog.com/support-lawfare
   INFO:root:rank=3 pagerank=8.4167e+00 url=www.lawfareblog.com/upcoming-events
   INFO:root:rank=4 pagerank=8.4167e+00 url=www.lawfareblog.com/our-comments-policy
   INFO:root:rank=5 pagerank=8.4167e+00 url=www.lawfareblog.com/subscribe-lawfare
   INFO:root:rank=6 pagerank=8.4167e+00 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban
   INFO:root:rank=7 pagerank=8.4167e+00 url=www.lawfareblog.com/cyberlaw-podcast-sandworm-and-grus-global-intifada
   INFO:root:rank=8 pagerank=8.4167e+00 url=www.lawfareblog.com/cyberlaw-podcast-plumbing-depths-artificial-stupidity
   INFO:root:rank=9 pagerank=8.4167e+00 url=www.lawfareblog.com/doj-charges-two-former-twitter-employees-spying-saudis

   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2
   INFO:gensim.models.keyedvectors:loading projection weights from /home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz
   INFO:gensim.utils:KeyedVectors lifecycle event {'msg': 'loaded (3000000, 300) matrix of type float32 from /home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz', 'binary': True, 'encoding': 'utf8', 'datetime': '2022-12-11T20:40:41.244681', 'gensim': '4.2.0', 'python': '3.6.9 (default, Dec  8 2021, 21:08:43) \n[GCC 8.4.0]', 'platform': 'Linux-4.15.0-88-generic-x86_64-with-Ubuntu-18.04-bionic', 'event': 'load_word2vec_format'}
   INFO:root:rank=0 pagerank=4.6091e+00 url=www.lawfareblog.com/0-days-n-days-iphones-and-android
   INFO:root:rank=1 pagerank=2.9867e+00 url=www.lawfareblog.com/lawfare-job-board
   INFO:root:rank=2 pagerank=2.9669e+00 url=www.lawfareblog.com/doj-charges-two-former-twitter-employees-spying-saudis
   INFO:root:rank=3 pagerank=2.0173e+00 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban
   INFO:root:rank=4 pagerank=1.8769e+00 url=www.lawfareblog.com/subscribe-lawfare
   INFO:root:rank=5 pagerank=1.8762e+00 url=www.lawfareblog.com/lessons-so-far-whatsapp-v-nso
   INFO:root:rank=6 pagerank=1.8693e+00 url=www.lawfareblog.com/cyberlaw-podcast-plumbing-depths-artificial-stupidity
   INFO:root:rank=7 pagerank=1.7655e+00 url=www.lawfareblog.com/our-comments-policy
   INFO:root:rank=8 pagerank=1.6807e+00 url=www.lawfareblog.com/upcoming-events
   INFO:root:rank=9 pagerank=9.8345e-01 url=www.lawfareblog.com/cyberlaw-podcast-sandworm-and-grus-global-intifada

   ```

   Task 1, part 4:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose 
   INFO:gensim.models.keyedvectors:loading projection weights from /home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz
   DEBUG:smart_open.smart_open_lib:{'transport_params': None, 'compression': 'infer_from_extension', 'opener': None, 'closefd': True, 'newline': None, 'errors': None, 'encoding': None, 'buffering': -1, 'mode': 'rb', 'uri': '/home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz'}
   DEBUG:gensim.utils:starting a new internal lifecycle event log for KeyedVectors
   INFO:gensim.utils:KeyedVectors lifecycle event {'msg': 'loaded (3000000, 300) matrix of type float32 from /home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz', 'binary': True, 'encoding': 'utf8', 'datetime': '2022-12-11T20:43:57.408185', 'gensim': '4.2.0', 'python': '3.6.9 (default, Dec  8 2021, 21:08:43) \n[GCC 8.4.0]', 'platform': 'Linux-4.15.0-88-generic-x86_64-with-Ubuntu-18.04-bionic', 'event': 'load_word2vec_format'}
   DEBUG:root:computing indices
   DEBUG:root:computing values
   DEBUG:root:i=0 residual=20.521432645835954
   DEBUG:root:i=1 residual=6.111089225341693
   DEBUG:root:i=2 residual=1.9220635359417255
   DEBUG:root:i=3 residual=0.5878388826079757
   DEBUG:root:i=4 residual=0.17554849198306322
   DEBUG:root:i=5 residual=0.051629178662097236
   DEBUG:root:i=6 residual=0.015030274556628363
   DEBUG:root:i=7 residual=0.004319490158400615
   DEBUG:root:i=8 residual=0.0011989704901153755
   DEBUG:root:i=9 residual=0.0002959382728162265
   DEBUG:root:i=10 residual=4.620655767044468e-05
   DEBUG:root:i=11 residual=3.908030230549799e-05
   DEBUG:root:i=12 residual=4.937970737112595e-05
   DEBUG:root:i=13 residual=4.7078161578568524e-05
   DEBUG:root:i=14 residual=4.155446446632934e-05
   DEBUG:root:i=15 residual=3.578187019538296e-05
   DEBUG:root:i=16 residual=3.055249267289263e-05
   DEBUG:root:i=17 residual=2.601096140064893e-05
   DEBUG:root:i=18 residual=2.212173188530955e-05
   DEBUG:root:i=19 residual=1.8807206721892782e-05
   DEBUG:root:i=20 residual=1.598725142658292e-05
   DEBUG:root:i=21 residual=1.3589503751510915e-05
   DEBUG:root:i=22 residual=1.1551181267355465e-05
   DEBUG:root:i=23 residual=9.818535427361315e-06
   DEBUG:root:i=24 residual=8.345764873396828e-06
   DEBUG:root:i=25 residual=7.093903258142301e-06
   DEBUG:root:i=26 residual=6.029818803882923e-06
   DEBUG:root:i=27 residual=5.125346388794453e-06
   DEBUG:root:i=28 residual=4.35654465865797e-06
   DEBUG:root:i=29 residual=3.7030631103961344e-06
   DEBUG:root:i=30 residual=3.1476037743344582e-06
   DEBUG:root:i=31 residual=2.6754632836062193e-06
   DEBUG:root:i=32 residual=2.2741438633871126e-06
   DEBUG:root:i=33 residual=1.933022333435052e-06
   DEBUG:root:i=34 residual=1.6430690292728054e-06
   DEBUG:root:i=35 residual=1.3966087145851132e-06
   DEBUG:root:i=36 residual=1.187117427716237e-06
   DEBUG:root:i=37 residual=1.0090498643213927e-06
   DEBUG:root:i=38 residual=8.576924089956801e-07
   INFO:root:rank=0 pagerank=8.4167e+00 url=www.lawfareblog.com/lawfare-job-board
   INFO:root:rank=1 pagerank=8.4167e+00 url=www.lawfareblog.com/snowden-revelations
   INFO:root:rank=2 pagerank=8.4167e+00 url=www.lawfareblog.com/support-lawfare
   INFO:root:rank=3 pagerank=8.4167e+00 url=www.lawfareblog.com/upcoming-events
   INFO:root:rank=4 pagerank=8.4167e+00 url=www.lawfareblog.com/our-comments-policy
   INFO:root:rank=5 pagerank=8.4167e+00 url=www.lawfareblog.com/subscribe-lawfare
   INFO:root:rank=6 pagerank=8.4167e+00 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban
   INFO:root:rank=7 pagerank=8.4167e+00 url=www.lawfareblog.com/cyberlaw-podcast-sandworm-and-grus-global-intifada
   INFO:root:rank=8 pagerank=8.4167e+00 url=www.lawfareblog.com/cyberlaw-podcast-plumbing-depths-artificial-stupidity
   INFO:root:rank=9 pagerank=8.4167e+00 url=www.lawfareblog.com/doj-charges-two-former-twitter-employees-spying-saudis



   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --alpha=0.99999
   INFO:gensim.models.keyedvectors:loading projection weights from /home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz
   DEBUG:smart_open.smart_open_lib:{'transport_params': None, 'compression': 'infer_from_extension', 'opener': None, 'closefd': True, 'newline': None, 'errors': None, 'encoding': None, 'buffering': -1, 'mode': 'rb', 'uri': '/home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz'}
   DEBUG:gensim.utils:starting a new internal lifecycle event log for KeyedVectors
   INFO:gensim.utils:KeyedVectors lifecycle event {'msg': 'loaded (3000000, 300) matrix of type float32 from /home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz', 'binary': True, 'encoding': 'utf8', 'datetime': '2022-12-11T20:46:29.984584', 'gensim': '4.2.0', 'python': '3.6.9 (default, Dec  8 2021, 21:08:43) \n[GCC 8.4.0]', 'platform': 'Linux-4.15.0-88-generic-x86_64-with-Ubuntu-18.04-bionic', 'event': 'load_word2vec_format'}
   DEBUG:root:computing indices
   DEBUG:root:computing values
   DEBUG:root:i=0 residual=24.142618822095447
   DEBUG:root:i=1 residual=8.458389858800029
   DEBUG:root:i=2 residual=3.130079789312069
   DEBUG:root:i=3 residual=1.1265218232333734
   DEBUG:root:i=4 residual=0.3960850532216708
   DEBUG:root:i=5 residual=0.13734921380387866
   DEBUG:root:i=6 residual=0.0473450832705688
   DEBUG:root:i=7 residual=0.016312073842668782
   DEBUG:root:i=8 residual=0.005634442408705543
   DEBUG:root:i=9 residual=0.001953840344974479
   DEBUG:root:i=10 residual=0.0006805793704147366
   DEBUG:root:i=11 residual=0.000238339202451431
   DEBUG:root:i=12 residual=8.415409539655682e-05
   DEBUG:root:i=13 residual=3.022057638009597e-05
   DEBUG:root:i=14 residual=1.130441331210929e-05
   DEBUG:root:i=15 residual=4.660826992454344e-06
   DEBUG:root:i=16 residual=2.333284789105125e-06
   DEBUG:root:i=17 residual=1.5243827941433434e-06
   DEBUG:root:i=18 residual=1.2444404793060062e-06
   DEBUG:root:i=19 residual=1.1468808633360249e-06
   DEBUG:root:i=20 residual=1.112553283649285e-06
   DEBUG:root:i=21 residual=1.1003935013724636e-06
   DEBUG:root:i=22 residual=1.0960650470001772e-06
   DEBUG:root:i=23 residual=1.0945158071520887e-06
   DEBUG:root:i=24 residual=1.0939554379017537e-06
   DEBUG:root:i=25 residual=1.0937478292139916e-06
   DEBUG:root:i=26 residual=1.0936664518475521e-06
   DEBUG:root:i=27 residual=1.093630170228314e-06
   DEBUG:root:i=28 residual=1.0936101062032526e-06
   DEBUG:root:i=29 residual=1.0935958755472008e-06
   DEBUG:root:i=30 residual=1.093583753760027e-06
   DEBUG:root:i=31 residual=1.093572408892921e-06
   DEBUG:root:i=32 residual=1.0935613722596085e-06
   DEBUG:root:i=33 residual=1.0935504281577268e-06
   DEBUG:root:i=34 residual=1.0935395025995322e-06
   DEBUG:root:i=35 residual=1.093528540139918e-06
   DEBUG:root:i=36 residual=1.093517645344323e-06
   DEBUG:root:i=37 residual=1.0935067751849743e-06
   DEBUG:root:i=38 residual=1.0934958619823817e-06
   DEBUG:root:i=39 residual=1.0934849733589126e-06
   DEBUG:root:i=40 residual=1.0934740232355578e-06
   DEBUG:root:i=41 residual=1.0934631407532547e-06
   DEBUG:root:i=42 residual=1.0934522706051378e-06
   DEBUG:root:i=43 residual=1.0934413266365865e-06
   DEBUG:root:i=44 residual=1.0934304257016375e-06
   DEBUG:root:i=45 residual=1.093419518629971e-06
   DEBUG:root:i=46 residual=1.0934086115585161e-06
   DEBUG:root:i=47 residual=1.0933977167932868e-06
   DEBUG:root:i=48 residual=1.0933868650901867e-06
   DEBUG:root:i=49 residual=1.0933759272917633e-06
   DEBUG:root:i=50 residual=1.093365014051099e-06
   DEBUG:root:i=51 residual=1.0933541438895137e-06
   DEBUG:root:i=52 residual=1.093343242985766e-06
   DEBUG:root:i=53 residual=1.093332329763123e-06
   DEBUG:root:i=54 residual=1.0933214473014594e-06
   DEBUG:root:i=55 residual=1.0933105525442364e-06
   DEBUG:root:i=56 residual=1.093299657781395e-06
   DEBUG:root:i=57 residual=1.093288738414691e-06
   DEBUG:root:i=58 residual=1.0932778313358229e-06
   DEBUG:root:i=59 residual=1.0932669181126505e-06
   DEBUG:root:i=60 residual=1.0932560294941802e-06
   DEBUG:root:i=61 residual=1.093245140885679e-06
   DEBUG:root:i=62 residual=1.093234215372264e-06
   DEBUG:root:i=63 residual=1.0932233144397018e-06
   DEBUG:root:i=64 residual=1.0932124135175981e-06
   DEBUG:root:i=65 residual=1.0932015249116313e-06
   DEBUG:root:i=66 residual=1.0931906239975786e-06
   DEBUG:root:i=67 residual=1.0931797107799541e-06
   DEBUG:root:i=68 residual=1.0931688590713127e-06
   DEBUG:root:i=69 residual=1.0931579827876634e-06
   DEBUG:root:i=70 residual=1.0931470572774445e-06
   DEBUG:root:i=71 residual=1.0931361686511844e-06
   DEBUG:root:i=72 residual=1.0931252677394033e-06
   DEBUG:root:i=73 residual=1.0931143606761825e-06
   DEBUG:root:i=74 residual=1.0931034966642377e-06
   DEBUG:root:i=75 residual=1.0930926019174999e-06
   DEBUG:root:i=76 residual=1.0930816579437355e-06
   DEBUG:root:i=77 residual=1.0930707631583616e-06
   DEBUG:root:i=78 residual=1.0930598560919956e-06
   DEBUG:root:i=79 residual=1.0930489674729195e-06
   DEBUG:root:i=80 residual=1.0930380788671923e-06
   DEBUG:root:i=81 residual=1.0930271902592403e-06
   DEBUG:root:i=82 residual=1.0930162893527008e-06
   DEBUG:root:i=83 residual=1.0930053515234363e-06
   DEBUG:root:i=84 residual=1.0929944505879465e-06
   DEBUG:root:i=85 residual=1.092983568128306e-06
   DEBUG:root:i=86 residual=1.0929726795228021e-06
   DEBUG:root:i=87 residual=1.092961803218436e-06
   DEBUG:root:i=88 residual=1.0929509269265971e-06
   DEBUG:root:i=89 residual=1.0929400075626784e-06
   DEBUG:root:i=90 residual=1.0929291189444209e-06
   DEBUG:root:i=91 residual=1.0929181995799361e-06
   DEBUG:root:i=92 residual=1.0929072740433379e-06
   DEBUG:root:i=93 residual=1.0928964284813605e-06
   DEBUG:root:i=94 residual=1.092885546045582e-06
   DEBUG:root:i=95 residual=1.092874614388673e-06
   DEBUG:root:i=96 residual=1.0928637257545289e-06
   DEBUG:root:i=97 residual=1.0928528433006932e-06
   DEBUG:root:i=98 residual=1.0928419608467036e-06
   DEBUG:root:i=99 residual=1.0928310599431691e-06
   DEBUG:root:i=100 residual=1.0928201221137384e-06
   DEBUG:root:i=101 residual=1.0928092396360664e-06
   DEBUG:root:i=102 residual=1.0927983448756037e-06
   DEBUG:root:i=103 residual=1.092787462422482e-06
   DEBUG:root:i=104 residual=1.0927765861200567e-06
   DEBUG:root:i=105 residual=1.0927656913706854e-06
   DEBUG:root:i=106 residual=1.092754778153484e-06
   DEBUG:root:i=107 residual=1.0927438956837714e-06
   DEBUG:root:i=108 residual=1.0927329947823698e-06
   DEBUG:root:i=109 residual=1.0927220877143839e-06
   DEBUG:root:i=110 residual=1.092711180637931e-06
   DEBUG:root:i=111 residual=1.0927002920216463e-06
   DEBUG:root:i=112 residual=1.092689440328885e-06
   DEBUG:root:i=113 residual=1.0926785271346645e-06
   DEBUG:root:i=114 residual=1.092667638514215e-06
   DEBUG:root:i=115 residual=1.0926567437540737e-06
   DEBUG:root:i=116 residual=1.0926458059324577e-06
   DEBUG:root:i=117 residual=1.092634941910351e-06
   DEBUG:root:i=118 residual=1.0926240410112113e-06
   DEBUG:root:i=119 residual=1.092613127791167e-06
   DEBUG:root:i=120 residual=1.0926022699312003e-06
   DEBUG:root:i=121 residual=1.0925913628806317e-06
   DEBUG:root:i=122 residual=1.0925804496634274e-06
   DEBUG:root:i=123 residual=1.092569554893089e-06
   DEBUG:root:i=124 residual=1.0925586724312475e-06
   DEBUG:root:i=125 residual=1.0925478022835478e-06
   DEBUG:root:i=126 residual=1.0925368952331577e-06
   DEBUG:root:i=127 residual=1.0925259943116334e-06
   DEBUG:root:i=128 residual=1.0925151180041648e-06
   DEBUG:root:i=129 residual=1.092504229403911e-06
   DEBUG:root:i=130 residual=1.0924933592536354e-06
   DEBUG:root:i=131 residual=1.092482470656201e-06
   DEBUG:root:i=132 residual=1.0924715574442959e-06
   DEBUG:root:i=133 residual=1.092460638066969e-06
   DEBUG:root:i=134 residual=1.0924497617465415e-06
   DEBUG:root:i=135 residual=1.0924388669970227e-06
   DEBUG:root:i=136 residual=1.0924280091421454e-06
   DEBUG:root:i=137 residual=1.0924170959482517e-06
   DEBUG:root:i=138 residual=1.0924062196333758e-06
   DEBUG:root:i=139 residual=1.0923953371795907e-06
   DEBUG:root:i=140 residual=1.0923844239776976e-06
   DEBUG:root:i=141 residual=1.0923735353542448e-06
   DEBUG:root:i=142 residual=1.0923626405966734e-06
   DEBUG:root:i=143 residual=1.0923517519863607e-06
   DEBUG:root:i=144 residual=1.0923408695370697e-06
   DEBUG:root:i=145 residual=1.0923299501682048e-06
   DEBUG:root:i=146 residual=1.0923190738533816e-06
   DEBUG:root:i=147 residual=1.0923081975537113e-06
   DEBUG:root:i=148 residual=1.0922973458663473e-06
   DEBUG:root:i=149 residual=1.0922864388266684e-06
   DEBUG:root:i=150 residual=1.0922755194473771e-06
   DEBUG:root:i=151 residual=1.0922646308315036e-06
   DEBUG:root:i=152 residual=1.0922537176139827e-06
   DEBUG:root:i=153 residual=1.0922428413013336e-06
   DEBUG:root:i=154 residual=1.0922319588471685e-06
   DEBUG:root:i=155 residual=1.0922210702494494e-06
   DEBUG:root:i=156 residual=1.0922101754900927e-06
   DEBUG:root:i=157 residual=1.09219928072784e-06
   DEBUG:root:i=158 residual=1.0921883982685331e-06
   DEBUG:root:i=159 residual=1.092177497364773e-06
   DEBUG:root:i=160 residual=1.0921666272070674e-06
   DEBUG:root:i=161 residual=1.0921557447636509e-06
   DEBUG:root:i=162 residual=1.0921448500066028e-06
   DEBUG:root:i=163 residual=1.0921339552444204e-06
   DEBUG:root:i=164 residual=1.0921230850938657e-06
   DEBUG:root:i=165 residual=1.0921121841954542e-06
   DEBUG:root:i=166 residual=1.0921012955776185e-06
   DEBUG:root:i=167 residual=1.092090400822617e-06
   DEBUG:root:i=168 residual=1.0920795122120516e-06
   DEBUG:root:i=169 residual=1.0920685990022046e-06
   DEBUG:root:i=170 residual=1.0920577226819778e-06
   DEBUG:root:i=171 residual=1.0920468340814818e-06
   DEBUG:root:i=172 residual=1.0920359331752633e-06
   DEBUG:root:i=173 residual=1.0920250568652549e-06
   DEBUG:root:i=174 residual=1.0920141621108173e-06
   DEBUG:root:i=175 residual=1.0920032981124442e-06
   DEBUG:root:i=176 residual=1.0919924402787268e-06
   DEBUG:root:i=177 residual=1.0919815209281937e-06
   DEBUG:root:i=178 residual=1.091970638459031e-06
   DEBUG:root:i=179 residual=1.0919597436987796e-06
   DEBUG:root:i=180 residual=1.0919488550962426e-06
   DEBUG:root:i=181 residual=1.0919379418812314e-06
   DEBUG:root:i=182 residual=1.0919271024739398e-06
   DEBUG:root:i=183 residual=1.0919162200460988e-06
   DEBUG:root:i=184 residual=1.0919053437500128e-06
   DEBUG:root:i=185 residual=1.0918944366968962e-06
   DEBUG:root:i=186 residual=1.091883554228327e-06
   DEBUG:root:i=187 residual=1.0918726779311503e-06
   DEBUG:root:i=188 residual=1.0918617585723766e-06
   DEBUG:root:i=189 residual=1.0918509007102029e-06
   DEBUG:root:i=190 residual=1.091839999814281e-06
   DEBUG:root:i=191 residual=1.091829098897529e-06
   DEBUG:root:i=192 residual=1.0918182287394438e-06
   DEBUG:root:i=193 residual=1.0918073586046258e-06
   DEBUG:root:i=194 residual=1.0917964453952254e-06
   DEBUG:root:i=195 residual=1.0917855813864958e-06
   DEBUG:root:i=196 residual=1.0917746927885103e-06
   DEBUG:root:i=197 residual=1.091763798036983e-06
   DEBUG:root:i=198 residual=1.0917529094264547e-06
   DEBUG:root:i=199 residual=1.091742051577305e-06
   DEBUG:root:i=200 residual=1.0917311383800069e-06
   DEBUG:root:i=201 residual=1.0917202436131677e-06
   DEBUG:root:i=202 residual=1.0917093857584322e-06
   DEBUG:root:i=203 residual=1.0916984664093981e-06
   DEBUG:root:i=204 residual=1.0916876208510977e-06
   DEBUG:root:i=205 residual=1.0916767384206346e-06
   DEBUG:root:i=206 residual=1.0916658805770234e-06
   DEBUG:root:i=207 residual=1.0916549796814578e-06
   DEBUG:root:i=208 residual=1.0916440910737658e-06
   DEBUG:root:i=209 residual=1.0916332209181726e-06
   DEBUG:root:i=210 residual=1.0916223815397891e-06
   DEBUG:root:i=211 residual=1.0916114560474397e-06
   DEBUG:root:i=212 residual=1.0916005735763129e-06
   DEBUG:root:i=213 residual=1.0915897034284838e-06
   DEBUG:root:i=214 residual=1.0915788025274352e-06
   DEBUG:root:i=215 residual=1.0915679139115582e-06
   DEBUG:root:i=216 residual=1.091557025311712e-06
   DEBUG:root:i=217 residual=1.091546112096654e-06
   DEBUG:root:i=218 residual=1.0915352788407644e-06
   DEBUG:root:i=219 residual=1.0915243779605403e-06
   DEBUG:root:i=220 residual=1.0915134955020125e-06
   DEBUG:root:i=221 residual=1.0915026499588343e-06
   DEBUG:root:i=222 residual=1.0914917429245049e-06
   DEBUG:root:i=223 residual=1.091480860460986e-06
   DEBUG:root:i=224 residual=1.0914699780044276e-06
   DEBUG:root:i=225 residual=1.0914590771036795e-06
   DEBUG:root:i=226 residual=1.0914481761869295e-06
   DEBUG:root:i=227 residual=1.0914373367898544e-06
   DEBUG:root:i=228 residual=1.091426435904489e-06
   DEBUG:root:i=229 residual=1.0914155841996633e-06
   DEBUG:root:i=230 residual=1.091404695614659e-06
   DEBUG:root:i=231 residual=1.0913938131618955e-06
   DEBUG:root:i=232 residual=1.0913829676209598e-06
   DEBUG:root:i=233 residual=1.0913720421288026e-06
   DEBUG:root:i=234 residual=1.0913612027199116e-06
   DEBUG:root:i=235 residual=1.0913502895305035e-06
   DEBUG:root:i=236 residual=1.0913394255192767e-06
   DEBUG:root:i=237 residual=1.0913285676830266e-06
   DEBUG:root:i=238 residual=1.0913176544867067e-06
   DEBUG:root:i=239 residual=1.0913067720228875e-06
   DEBUG:root:i=240 residual=1.0912958834198378e-06
   DEBUG:root:i=241 residual=1.0912850071151668e-06
   DEBUG:root:i=242 residual=1.0912741062115635e-06
   DEBUG:root:i=243 residual=1.09126323605667e-06
   DEBUG:root:i=244 residual=1.0912523474563065e-06
   DEBUG:root:i=245 residual=1.0912414650052037e-06
   DEBUG:root:i=246 residual=1.0912306010093163e-06
   DEBUG:root:i=247 residual=1.0912197308696776e-06
   DEBUG:root:i=248 residual=1.0912088176707228e-06
   DEBUG:root:i=249 residual=1.0911979721146979e-06
   DEBUG:root:i=250 residual=1.0911870712236447e-06
   DEBUG:root:i=251 residual=1.0911761826137821e-06
   DEBUG:root:i=252 residual=1.0911653001600576e-06
   DEBUG:root:i=253 residual=1.091154430012241e-06
   DEBUG:root:i=254 residual=1.0911435352632982e-06
   DEBUG:root:i=255 residual=1.0911327020181333e-06
   DEBUG:root:i=256 residual=1.091121807294963e-06
   DEBUG:root:i=257 residual=1.0911109125305687e-06
   DEBUG:root:i=258 residual=1.0911000485299476e-06
   DEBUG:root:i=259 residual=1.0910891476339317e-06
   DEBUG:root:i=260 residual=1.091078283627774e-06
   DEBUG:root:i=261 residual=1.0910674257914753e-06
   DEBUG:root:i=262 residual=1.0910565618139774e-06
   DEBUG:root:i=263 residual=1.09104566091344e-06
   DEBUG:root:i=264 residual=1.091034821522294e-06
   DEBUG:root:i=265 residual=1.0910239206316725e-06
   DEBUG:root:i=266 residual=1.0910130320243228e-06
   DEBUG:root:i=267 residual=1.0910021495682431e-06
   DEBUG:root:i=268 residual=1.0909912794228603e-06
   DEBUG:root:i=269 residual=1.0909804461857624e-06
   DEBUG:root:i=270 residual=1.0909695145550073e-06
   DEBUG:root:i=271 residual=1.090958668986382e-06
   DEBUG:root:i=272 residual=1.0909477681007102e-06
   DEBUG:root:i=273 residual=1.0909368856368043e-06
   DEBUG:root:i=274 residual=1.0909260216463158e-06
   DEBUG:root:i=275 residual=1.0909151330513187e-06
   DEBUG:root:i=276 residual=1.0909042690478744e-06
   DEBUG:root:i=277 residual=1.0908933743091053e-06
   DEBUG:root:i=278 residual=1.0908825533678983e-06
   DEBUG:root:i=279 residual=1.0908716709482927e-06
   DEBUG:root:i=280 residual=1.0908607823486026e-06
   DEBUG:root:i=281 residual=1.090849887591994e-06
   DEBUG:root:i=282 residual=1.0908390051357096e-06
   DEBUG:root:i=283 residual=1.090828178047337e-06
   DEBUG:root:i=284 residual=1.0908172710183947e-06
   DEBUG:root:i=285 residual=1.0908064500746315e-06
   DEBUG:root:i=286 residual=1.0907955492001184e-06
   DEBUG:root:i=287 residual=1.0907846544408987e-06
   DEBUG:root:i=288 residual=1.0907738088925799e-06
   DEBUG:root:i=289 residual=1.0907628895465647e-06
   DEBUG:root:i=290 residual=1.0907520501425066e-06
   DEBUG:root:i=291 residual=1.0907411431026187e-06
   DEBUG:root:i=292 residual=1.0907302914028079e-06
   DEBUG:root:i=293 residual=1.0907193782086746e-06
   DEBUG:root:i=294 residual=1.0907085018916896e-06
   DEBUG:root:i=295 residual=1.0906976440496968e-06
   DEBUG:root:i=296 residual=1.0906867554624443e-06
   DEBUG:root:i=297 residual=1.0906759345265751e-06
   DEBUG:root:i=298 residual=1.0906650336518866e-06
   DEBUG:root:i=299 residual=1.090654145042137e-06
   DEBUG:root:i=300 residual=1.0906432995016528e-06
   DEBUG:root:i=301 residual=1.0906324109117465e-06
   DEBUG:root:i=302 residual=1.0906215346156106e-06
   DEBUG:root:i=303 residual=1.090610639861037e-06
   DEBUG:root:i=304 residual=1.0905997943125552e-06
   DEBUG:root:i=305 residual=1.0905889303405327e-06
   DEBUG:root:i=306 residual=1.0905780417459398e-06
   DEBUG:root:i=307 residual=1.0905671469919127e-06
   DEBUG:root:i=308 residual=1.0905562768365018e-06
   DEBUG:root:i=309 residual=1.0905453943904459e-06
   DEBUG:root:i=310 residual=1.0905345119396663e-06
   DEBUG:root:i=311 residual=1.09052368485697e-06
   DEBUG:root:i=312 residual=1.0905127901287609e-06
   DEBUG:root:i=313 residual=1.0905019322823972e-06
   DEBUG:root:i=314 residual=1.0904910621480874e-06
   DEBUG:root:i=315 residual=1.0904801858547911e-06
   DEBUG:root:i=316 residual=1.090469297254712e-06
   DEBUG:root:i=317 residual=1.090458433261232e-06
   DEBUG:root:i=318 residual=1.0904475692713648e-06
   DEBUG:root:i=319 residual=1.0904367114398905e-06
   DEBUG:root:i=320 residual=1.0904258474551888e-06
   DEBUG:root:i=321 residual=1.0904149896220811e-06
   DEBUG:root:i=322 residual=1.0904040641246217e-06
   DEBUG:root:i=323 residual=1.0903932124066994e-06
   DEBUG:root:i=324 residual=1.0903823607348775e-06
   DEBUG:root:i=325 residual=1.0903714844488035e-06
   DEBUG:root:i=326 residual=1.0903605958517728e-06
   DEBUG:root:i=327 residual=1.0903497626151694e-06
   DEBUG:root:i=328 residual=1.0903388863364315e-06
   DEBUG:root:i=329 residual=1.090327985438798e-06
   DEBUG:root:i=330 residual=1.0903171091268176e-06
   DEBUG:root:i=331 residual=1.0903062574395128e-06
   DEBUG:root:i=332 residual=1.0902953996134968e-06
   DEBUG:root:i=333 residual=1.0902845171737847e-06
   DEBUG:root:i=334 residual=1.0902736593324266e-06
   DEBUG:root:i=335 residual=1.09026275229032e-06
   DEBUG:root:i=336 residual=1.0902519251874488e-06
   DEBUG:root:i=337 residual=1.0902410366150445e-06
   DEBUG:root:i=338 residual=1.0902301787720285e-06
   DEBUG:root:i=339 residual=1.090219284030759e-06
   DEBUG:root:i=340 residual=1.090208450786145e-06
   DEBUG:root:i=341 residual=1.0901976052689394e-06
   DEBUG:root:i=342 residual=1.0901867043869395e-06
   DEBUG:root:i=343 residual=1.0901758649904035e-06
   DEBUG:root:i=344 residual=1.0901649641050485e-06
   DEBUG:root:i=345 residual=1.0901541001024843e-06
   DEBUG:root:i=346 residual=1.0901432422686603e-06
   DEBUG:root:i=347 residual=1.0901323536789002e-06
   DEBUG:root:i=348 residual=1.0901214896762892e-06
   DEBUG:root:i=349 residual=1.0901106379988057e-06
   DEBUG:root:i=350 residual=1.0900997740167098e-06
   DEBUG:root:i=351 residual=1.0900888546689946e-06
   DEBUG:root:i=352 residual=1.090078021413498e-06
   DEBUG:root:i=353 residual=1.0900671451349274e-06
   DEBUG:root:i=354 residual=1.0900562442372682e-06
   DEBUG:root:i=355 residual=1.0900454232987081e-06
   DEBUG:root:i=356 residual=1.0900345470249375e-06
   DEBUG:root:i=357 residual=1.09002367688632e-06
   DEBUG:root:i=358 residual=1.0900127944408586e-06
   DEBUG:root:i=359 residual=1.090001961203776e-06
   DEBUG:root:i=360 residual=1.0899910910825064e-06
   DEBUG:root:i=361 residual=1.0899802147889129e-06
   DEBUG:root:i=362 residual=1.0899693384976788e-06
   DEBUG:root:i=363 residual=1.0899584745049073e-06
   DEBUG:root:i=364 residual=1.0899475982161334e-06
   DEBUG:root:i=365 residual=1.0899367342209005e-06
   DEBUG:root:i=366 residual=1.089925864086568e-06
   DEBUG:root:i=367 residual=1.0899150062504934e-06
   DEBUG:root:i=368 residual=1.0899041238123456e-06
   DEBUG:root:i=369 residual=1.0898932598172078e-06
   DEBUG:root:i=370 residual=1.089882389677833e-06
   DEBUG:root:i=371 residual=1.0898715072373274e-06
   DEBUG:root:i=372 residual=1.0898606432393364e-06
   DEBUG:root:i=373 residual=1.0898498038635444e-06
   DEBUG:root:i=374 residual=1.0898389275850272e-06
   DEBUG:root:i=375 residual=1.0898280820505513e-06
   DEBUG:root:i=376 residual=1.0898171934665317e-06
   DEBUG:root:i=377 residual=1.0898063417741133e-06
   DEBUG:root:i=378 residual=1.0897954716428276e-06
   DEBUG:root:i=379 residual=1.0897846015010724e-06
   DEBUG:root:i=380 residual=1.0897737498191155e-06
   DEBUG:root:i=381 residual=1.089762898140009e-06
   DEBUG:root:i=382 residual=1.0897520218620998e-06
   DEBUG:root:i=383 residual=1.0897411578643379e-06
   DEBUG:root:i=384 residual=1.0897303246428142e-06
   DEBUG:root:i=385 residual=1.0897194360643022e-06
   DEBUG:root:i=386 residual=1.0897085843719137e-06
   DEBUG:root:i=387 residual=1.0896977203951948e-06
   DEBUG:root:i=388 residual=1.0896868256463957e-06
   DEBUG:root:i=389 residual=1.0896760231678365e-06
   DEBUG:root:i=390 residual=1.0896650915424732e-06
   DEBUG:root:i=391 residual=1.0896542521283666e-06
   DEBUG:root:i=392 residual=1.0896433881556764e-06
   DEBUG:root:i=393 residual=1.0896325057181016e-06
   DEBUG:root:i=394 residual=1.089621697088108e-06
   DEBUG:root:i=395 residual=1.0896107962163518e-06
   DEBUG:root:i=396 residual=1.0895999199125509e-06
   DEBUG:root:i=397 residual=1.0895890682254758e-06
   DEBUG:root:i=398 residual=1.0895782103971046e-06
   DEBUG:root:i=399 residual=1.0895673464148933e-06
   DEBUG:root:i=400 residual=1.089556476275443e-06
   DEBUG:root:i=401 residual=1.0895456615041841e-06
   DEBUG:root:i=402 residual=1.089534754483584e-06
   DEBUG:root:i=403 residual=1.0895238781719084e-06
   DEBUG:root:i=404 residual=1.0895129895741068e-06
   DEBUG:root:i=405 residual=1.0895021255733764e-06
   DEBUG:root:i=406 residual=1.0894912677369238e-06
   DEBUG:root:i=407 residual=1.0894803853046176e-06
   DEBUG:root:i=408 residual=1.0894695582173294e-06
   DEBUG:root:i=409 residual=1.0894586942497133e-06
   DEBUG:root:i=410 residual=1.089447842571225e-06
   DEBUG:root:i=411 residual=1.0894369847408838e-06
   DEBUG:root:i=412 residual=1.0894261392188646e-06
   DEBUG:root:i=413 residual=1.0894152813959912e-06
   DEBUG:root:i=414 residual=1.0894043989537576e-06
   DEBUG:root:i=415 residual=1.0893935411150608e-06
   DEBUG:root:i=416 residual=1.0893826709807938e-06
   DEBUG:root:i=417 residual=1.0893718316026712e-06
   DEBUG:root:i=418 residual=1.0893609553195405e-06
   DEBUG:root:i=419 residual=1.089350091332067e-06
   DEBUG:root:i=420 residual=1.089339245802244e-06
   DEBUG:root:i=421 residual=1.0893283572203964e-06
   DEBUG:root:i=422 residual=1.0893175362845819e-06
   DEBUG:root:i=423 residual=1.089306641561852e-06
   DEBUG:root:i=424 residual=1.0892957898649383e-06
   DEBUG:root:i=425 residual=1.0892849258849467e-06
   DEBUG:root:i=426 residual=1.0892740742060453e-06
   DEBUG:root:i=427 residual=1.0892631979202317e-06
   DEBUG:root:i=428 residual=1.0892523400815024e-06
   DEBUG:root:i=429 residual=1.0892414699527347e-06
   DEBUG:root:i=430 residual=1.089230630564373e-06
   DEBUG:root:i=431 residual=1.0892197542939956e-06
   DEBUG:root:i=432 residual=1.0892088964504188e-06
   DEBUG:root:i=433 residual=1.0891980201643785e-06
   DEBUG:root:i=434 residual=1.0891871930897877e-06
   DEBUG:root:i=435 residual=1.0891763352695489e-06
   DEBUG:root:i=436 residual=1.0891654774418097e-06
   DEBUG:root:i=437 residual=1.0891546257653516e-06
   DEBUG:root:i=438 residual=1.0891437679377844e-06
   DEBUG:root:i=439 residual=1.0891328608906465e-06
   DEBUG:root:i=440 residual=1.0891220276461367e-06
   DEBUG:root:i=441 residual=1.0891111759770248e-06
   DEBUG:root:i=442 residual=1.0891002996943642e-06
   DEBUG:root:i=443 residual=1.089089497226426e-06
   DEBUG:root:i=444 residual=1.089078584053719e-06
   DEBUG:root:i=445 residual=1.089067775411457e-06
   DEBUG:root:i=446 residual=1.0890568622408243e-06
   DEBUG:root:i=447 residual=1.0890459982304055e-06
   DEBUG:root:i=448 residual=1.0890351342449527e-06
   DEBUG:root:i=449 residual=1.089024294868877e-06
   DEBUG:root:i=450 residual=1.0890134185884974e-06
   DEBUG:root:i=451 residual=1.0890025915089699e-06
   DEBUG:root:i=452 residual=1.0889917029375694e-06
   DEBUG:root:i=453 residual=1.0889808512434584e-06
   DEBUG:root:i=454 residual=1.0889699934151761e-06
   DEBUG:root:i=455 residual=1.088959147891066e-06
   DEBUG:root:i=456 residual=1.0889482654555622e-06
   DEBUG:root:i=457 residual=1.0889374752865726e-06
   DEBUG:root:i=458 residual=1.0889265744300313e-06
   DEBUG:root:i=459 residual=1.0889156858207816e-06
   DEBUG:root:i=460 residual=1.0889048771854116e-06
   DEBUG:root:i=461 residual=1.0888939824701976e-06
   DEBUG:root:i=462 residual=1.0888831676838109e-06
   DEBUG:root:i=463 residual=1.0888722975728404e-06
   DEBUG:root:i=464 residual=1.0888614581898436e-06
   DEBUG:root:i=465 residual=1.0888505757658555e-06
   DEBUG:root:i=466 residual=1.088839705621317e-06
   DEBUG:root:i=467 residual=1.088828909297144e-06
   DEBUG:root:i=468 residual=1.0888180391969023e-06
   DEBUG:root:i=469 residual=1.0888071567576036e-06
   DEBUG:root:i=470 residual=1.0887963481276495e-06
   DEBUG:root:i=471 residual=1.0887854657137211e-06
   DEBUG:root:i=472 residual=1.0887745955719833e-06
   DEBUG:root:i=473 residual=1.088763750036665e-06
   DEBUG:root:i=474 residual=1.0887528860596733e-06
   DEBUG:root:i=475 residual=1.0887420405328116e-06
   DEBUG:root:i=476 residual=1.0887311765551266e-06
   DEBUG:root:i=477 residual=1.0887203248767766e-06
   DEBUG:root:i=478 residual=1.088709454745208e-06
   DEBUG:root:i=479 residual=1.088698621513695e-06
   DEBUG:root:i=480 residual=1.0886877637012368e-06
   DEBUG:root:i=481 residual=1.088676881261585e-06
   DEBUG:root:i=482 residual=1.0886660726318796e-06
   DEBUG:root:i=483 residual=1.0886551840706382e-06
   DEBUG:root:i=484 residual=1.0886443754391055e-06
   DEBUG:root:i=485 residual=1.0886334868729337e-06
   DEBUG:root:i=486 residual=1.0886226597937234e-06
   DEBUG:root:i=487 residual=1.0886117712123913e-06
   DEBUG:root:i=488 residual=1.0886009625848338e-06
   DEBUG:root:i=489 residual=1.0885901293872674e-06
   DEBUG:root:i=490 residual=1.088579271570595e-06
   DEBUG:root:i=491 residual=1.0885683891356265e-06
   DEBUG:root:i=492 residual=1.0885575682025045e-06
   DEBUG:root:i=493 residual=1.0885467042413805e-06
   DEBUG:root:i=494 residual=1.0885358341044176e-06
   DEBUG:root:i=495 residual=1.0885249947238287e-06
   DEBUG:root:i=496 residual=1.0885141492046953e-06
   DEBUG:root:i=497 residual=1.0885032790812014e-06
   DEBUG:root:i=498 residual=1.088492427392248e-06
   DEBUG:root:i=499 residual=1.0884815388104096e-06
   DEBUG:root:i=500 residual=1.0884707178745937e-06
   DEBUG:root:i=501 residual=1.0884598908209103e-06
   DEBUG:root:i=502 residual=1.0884490207050429e-06
   DEBUG:root:i=503 residual=1.0884381813252892e-06
   DEBUG:root:i=504 residual=1.088427298895312e-06
   DEBUG:root:i=505 residual=1.088416453360566e-06
   DEBUG:root:i=506 residual=1.088405589383207e-06
   DEBUG:root:i=507 residual=1.0883947376993658e-06
   DEBUG:root:i=508 residual=1.0883839044835377e-06
   DEBUG:root:i=509 residual=1.088373071269993e-06
   DEBUG:root:i=510 residual=1.088362195000361e-06
   DEBUG:root:i=511 residual=1.0883513494656505e-06
   DEBUG:root:i=512 residual=1.0883404854886173e-06
   DEBUG:root:i=513 residual=1.0883296461106246e-06
   DEBUG:root:i=514 residual=1.0883187390793127e-06
   DEBUG:root:i=515 residual=1.0883079488916403e-06
   DEBUG:root:i=516 residual=1.0882970787938653e-06
   DEBUG:root:i=517 residual=1.0882862209558348e-06
   DEBUG:root:i=518 residual=1.0882753877323087e-06
   DEBUG:root:i=519 residual=1.088264548372606e-06
   DEBUG:root:i=520 residual=1.0882537090076846e-06
   DEBUG:root:i=521 residual=1.0882428450387786e-06
   DEBUG:root:i=522 residual=1.088231999506829e-06
   DEBUG:root:i=523 residual=1.0882211416870235e-06
   DEBUG:root:i=524 residual=1.088210296156876e-06
   DEBUG:root:i=525 residual=1.0881994383344769e-06
   DEBUG:root:i=526 residual=1.0881886112650615e-06
   DEBUG:root:i=527 residual=1.0881777165399728e-06
   DEBUG:root:i=528 residual=1.0881668894498781e-06
   DEBUG:root:i=529 residual=1.0881560254831817e-06
   DEBUG:root:i=530 residual=1.0881451922543822e-06
   DEBUG:root:i=531 residual=1.0881343405936302e-06
   DEBUG:root:i=532 residual=1.088123501226445e-06
   DEBUG:root:i=533 residual=1.0881126310981133e-06
   DEBUG:root:i=534 residual=1.0881018163243836e-06
   DEBUG:root:i=535 residual=1.0880909400620328e-06
   DEBUG:root:i=536 residual=1.0880801252888164e-06
   DEBUG:root:i=537 residual=1.088069310538372e-06
   DEBUG:root:i=538 residual=1.0880584096695772e-06
   DEBUG:root:i=539 residual=1.0880475579735249e-06
   DEBUG:root:i=540 residual=1.0880367063020775e-06
   DEBUG:root:i=541 residual=1.088025830013494e-06
   DEBUG:root:i=542 residual=1.088015058299048e-06
   DEBUG:root:i=543 residual=1.0880041882124085e-06
   DEBUG:root:i=544 residual=1.0879933303769753e-06
   DEBUG:root:i=545 residual=1.0879824787012206e-06
   DEBUG:root:i=546 residual=1.0879716454772969e-06
   DEBUG:root:i=547 residual=1.0879607999659576e-06
   DEBUG:root:i=548 residual=1.087949948298005e-06
   DEBUG:root:i=549 residual=1.0879390843159097e-06
   DEBUG:root:i=550 residual=1.0879282633984516e-06
   DEBUG:root:i=551 residual=1.0879174117349878e-06
   DEBUG:root:i=552 residual=1.0879065539104087e-06
   DEBUG:root:i=553 residual=1.087895726843584e-06
   DEBUG:root:i=554 residual=1.0878848628718954e-06
   DEBUG:root:i=555 residual=1.0878740358027777e-06
   DEBUG:root:i=556 residual=1.0878632025895118e-06
   DEBUG:root:i=557 residual=1.0878523632352798e-06
   DEBUG:root:i=558 residual=1.0878414992641223e-06
   DEBUG:root:i=559 residual=1.0878306783414988e-06
   DEBUG:root:i=560 residual=1.0878198082258633e-06
   DEBUG:root:i=561 residual=1.0878089319350836e-06
   DEBUG:root:i=562 residual=1.087798123310419e-06
   DEBUG:root:i=563 residual=1.0877872716551314e-06
   DEBUG:root:i=564 residual=1.0877764199842443e-06
   DEBUG:root:i=565 residual=1.087765586763399e-06
   DEBUG:root:i=566 residual=1.0877547104912141e-06
   DEBUG:root:i=567 residual=1.0877438957172837e-06
   DEBUG:root:i=568 residual=1.0877330317587408e-06
   DEBUG:root:i=569 residual=1.0877222046819473e-06
   DEBUG:root:i=570 residual=1.0877113714766132e-06
   DEBUG:root:i=571 residual=1.0877004890498866e-06
   DEBUG:root:i=572 residual=1.0876896619705361e-06
   DEBUG:root:i=573 residual=1.0876788226104408e-06
   DEBUG:root:i=574 residual=1.0876679340346605e-06
   DEBUG:root:i=575 residual=1.0876571315544004e-06
   DEBUG:root:i=576 residual=1.087646298356721e-06
   DEBUG:root:i=577 residual=1.0876354405445623e-06
   DEBUG:root:i=578 residual=1.0876245827117063e-06
   DEBUG:root:i=579 residual=1.0876137494909924e-06
   DEBUG:root:i=580 residual=1.087602916280563e-06
   DEBUG:root:i=581 residual=1.087592040010236e-06
   DEBUG:root:i=582 residual=1.0875812006302033e-06
   DEBUG:root:i=583 residual=1.0875704166233754e-06
   DEBUG:root:i=584 residual=1.0875595588369196e-06
   DEBUG:root:i=585 residual=1.0875487071589488e-06
   DEBUG:root:i=586 residual=1.0875378431846628e-06
   DEBUG:root:i=587 residual=1.0875270468662426e-06
   DEBUG:root:i=588 residual=1.0875161644626025e-06
   DEBUG:root:i=589 residual=1.0875053189234393e-06
   DEBUG:root:i=590 residual=1.087494498011113e-06
   DEBUG:root:i=591 residual=1.0874836463501585e-06
   DEBUG:root:i=592 residual=1.0874728008337498e-06
   DEBUG:root:i=593 residual=1.0874619553125356e-06
   DEBUG:root:i=594 residual=1.087451103638796e-06
   DEBUG:root:i=595 residual=1.0874402704207839e-06
   DEBUG:root:i=596 residual=1.087429418755093e-06
   DEBUG:root:i=597 residual=1.0874185916884848e-06
   DEBUG:root:i=598 residual=1.0874077400282383e-06
   DEBUG:root:i=599 residual=1.0873969006559418e-06
   DEBUG:root:i=600 residual=1.0873860674424529e-06
   DEBUG:root:i=601 residual=1.0873752342350037e-06
   DEBUG:root:i=602 residual=1.08736438256914e-06
   DEBUG:root:i=603 residual=1.0873535308965648e-06
   DEBUG:root:i=604 residual=1.0873426792201242e-06
   DEBUG:root:i=605 residual=1.0873318583027803e-06
   DEBUG:root:i=606 residual=1.0873210251049729e-06
   DEBUG:root:i=607 residual=1.087310161131321e-06
   DEBUG:root:i=608 residual=1.0872993279131254e-06
   DEBUG:root:i=609 residual=1.087288488545468e-06
   DEBUG:root:i=610 residual=1.0872776368774642e-06
   DEBUG:root:i=611 residual=1.0872668221173798e-06
   DEBUG:root:i=612 residual=1.0872559458546886e-06
   DEBUG:root:i=613 residual=1.087245118773393e-06
   DEBUG:root:i=614 residual=1.0872342855677439e-06
   DEBUG:root:i=615 residual=1.0872234523600937e-06
   DEBUG:root:i=616 residual=1.0872126376021902e-06
   DEBUG:root:i=617 residual=1.0872017920992188e-06
   DEBUG:root:i=618 residual=1.0871909711897188e-06
   DEBUG:root:i=619 residual=1.0871800949277738e-06
   DEBUG:root:i=620 residual=1.087169255542808e-06
   DEBUG:root:i=621 residual=1.0871583792621299e-06
   DEBUG:root:i=622 residual=1.0871475767944798e-06
   DEBUG:root:i=623 residual=1.0871367559003284e-06
   DEBUG:root:i=624 residual=1.0871259042455952e-06
   DEBUG:root:i=625 residual=1.0871150710250352e-06
   DEBUG:root:i=626 residual=1.0871042316628553e-06
   DEBUG:root:i=627 residual=1.0870933984550313e-06
   DEBUG:root:i=628 residual=1.0870825652425965e-06
   DEBUG:root:i=629 residual=1.0870717074274917e-06
   DEBUG:root:i=630 residual=1.0870608742046092e-06
   DEBUG:root:i=631 residual=1.0870500656010537e-06
   DEBUG:root:i=632 residual=1.0870391893442284e-06
   DEBUG:root:i=633 residual=1.0870283499615933e-06
   DEBUG:root:i=634 residual=1.0870175167481073e-06
   DEBUG:root:i=635 residual=1.0870066958415297e-06
   DEBUG:root:i=636 residual=1.086995862638807e-06
   DEBUG:root:i=637 residual=1.0869850109762253e-06
   DEBUG:root:i=638 residual=1.0869741839047068e-06
   DEBUG:root:i=639 residual=1.0869633383987009e-06
   DEBUG:root:i=640 residual=1.0869524744244814e-06
   DEBUG:root:i=641 residual=1.0869416535020048e-06
   DEBUG:root:i=642 residual=1.0869308202966459e-06
   DEBUG:root:i=643 residual=1.0869199624847132e-06
   DEBUG:root:i=644 residual=1.0869091477137415e-06
   DEBUG:root:i=645 residual=1.0868983268178111e-06
   DEBUG:root:i=646 residual=1.086887481316826e-06
   DEBUG:root:i=647 residual=1.0868766788552045e-06
   DEBUG:root:i=648 residual=1.086865827207713e-06
   DEBUG:root:i=649 residual=1.0868549693829878e-06
   DEBUG:root:i=650 residual=1.0868441607720515e-06
   DEBUG:root:i=651 residual=1.0868333152660244e-06
   DEBUG:root:i=652 residual=1.0868224697494264e-06
   DEBUG:root:i=653 residual=1.086811642683395e-06
   DEBUG:root:i=654 residual=1.0868007848738845e-06
   DEBUG:root:i=655 residual=1.0867899885609834e-06
   DEBUG:root:i=656 residual=1.0867791369159937e-06
   DEBUG:root:i=657 residual=1.0867682913921891e-06
   DEBUG:root:i=658 residual=1.086757439721585e-06
   DEBUG:root:i=659 residual=1.0867466188092494e-06
   DEBUG:root:i=660 residual=1.0867358040568884e-06
   DEBUG:root:i=661 residual=1.0867249708622944e-06
   DEBUG:root:i=662 residual=1.0867141376497353e-06
   DEBUG:root:i=663 residual=1.0867032798373897e-06
   DEBUG:root:i=664 residual=1.0866924650671619e-06
   DEBUG:root:i=665 residual=1.0866816134145526e-06
   DEBUG:root:i=666 residual=1.0866707986490228e-06
   DEBUG:root:i=667 residual=1.086659934692944e-06
   DEBUG:root:i=668 residual=1.0866491076167296e-06
   DEBUG:root:i=669 residual=1.086638311324314e-06
   DEBUG:root:i=670 residual=1.0866274535250046e-06
   DEBUG:root:i=671 residual=1.0866165772430307e-06
   DEBUG:root:i=672 residual=1.0866057563152748e-06
   DEBUG:root:i=673 residual=1.0865949354129705e-06
   DEBUG:root:i=674 residual=1.0865841145142187e-06
   DEBUG:root:i=675 residual=1.0865732813117522e-06
   DEBUG:root:i=676 residual=1.0865624419523267e-06
   DEBUG:root:i=677 residual=1.086551602588212e-06
   DEBUG:root:i=678 residual=1.0865407570720014e-06
   DEBUG:root:i=679 residual=1.0865299607691143e-06
   DEBUG:root:i=680 residual=1.086519139880758e-06
   DEBUG:root:i=681 residual=1.086508263613724e-06
   DEBUG:root:i=682 residual=1.0864974426889903e-06
   DEBUG:root:i=683 residual=1.0864866156352672e-06
   DEBUG:root:i=684 residual=1.0864757762811318e-06
   DEBUG:root:i=685 residual=1.0864649615239095e-06
   DEBUG:root:i=686 residual=1.086454140627187e-06
   DEBUG:root:i=687 residual=1.0864433074253333e-06
   DEBUG:root:i=688 residual=1.0864324496082242e-06
   DEBUG:root:i=689 residual=1.0864216225365803e-06
   DEBUG:root:i=690 residual=1.0864107647244526e-06
   DEBUG:root:i=691 residual=1.0863999561109028e-06
   DEBUG:root:i=692 residual=1.0863891290674892e-06
   DEBUG:root:i=693 residual=1.0863782835541978e-06
   DEBUG:root:i=694 residual=1.08636746264495e-06
   DEBUG:root:i=695 residual=1.0863566479003016e-06
   DEBUG:root:i=696 residual=1.0863457962453772e-06
   DEBUG:root:i=697 residual=1.0863349753283714e-06
   DEBUG:root:i=698 residual=1.086324166730209e-06
   DEBUG:root:i=699 residual=1.0863133765976426e-06
   DEBUG:root:i=700 residual=1.086302506503539e-06
   DEBUG:root:i=701 residual=1.0862916978828459e-06
   DEBUG:root:i=702 residual=1.0862808708343505e-06
   DEBUG:root:i=703 residual=1.0862700314802157e-06
   DEBUG:root:i=704 residual=1.0862592167279396e-06
   DEBUG:root:i=705 residual=1.086248401978397e-06
   DEBUG:root:i=706 residual=1.0862375503260843e-06
   DEBUG:root:i=707 residual=1.0862266986505608e-06
   DEBUG:root:i=708 residual=1.0862158715866994e-06
   DEBUG:root:i=709 residual=1.0862050691350418e-06
   DEBUG:root:i=710 residual=1.0861942297911012e-06
   DEBUG:root:i=711 residual=1.0861833781313023e-06
   DEBUG:root:i=712 residual=1.086172575663997e-06
   DEBUG:root:i=713 residual=1.0861617178625593e-06
   DEBUG:root:i=714 residual=1.0861509092543337e-06
   DEBUG:root:i=715 residual=1.0861400699023442e-06
   DEBUG:root:i=716 residual=1.0861292428443176e-06
   DEBUG:root:i=717 residual=1.0861183973331713e-06
   DEBUG:root:i=718 residual=1.0861075764184764e-06
   DEBUG:root:i=719 residual=1.0860967309177365e-06
   DEBUG:root:i=720 residual=1.0860859161555187e-06
   DEBUG:root:i=721 residual=1.0860750952611325e-06
   DEBUG:root:i=722 residual=1.0860642497530331e-06
   DEBUG:root:i=723 residual=1.0860534472964627e-06
   DEBUG:root:i=724 residual=1.0860426079555418e-06
   DEBUG:root:i=725 residual=1.086031799346939e-06
   DEBUG:root:i=726 residual=1.0860209846076584e-06
   DEBUG:root:i=727 residual=1.0860101267986967e-06
   DEBUG:root:i=728 residual=1.085999299735035e-06
   DEBUG:root:i=729 residual=1.0859884849800504e-06
   DEBUG:root:i=730 residual=1.0859776517804108e-06
   DEBUG:root:i=731 residual=1.0859668247270355e-06
   DEBUG:root:i=732 residual=1.085955997671398e-06
   DEBUG:root:i=733 residual=1.0859451644640855e-06
   DEBUG:root:i=734 residual=1.0859343251072047e-06
   DEBUG:root:i=735 residual=1.08592351034717e-06
   DEBUG:root:i=736 residual=1.085912701756585e-06
   DEBUG:root:i=737 residual=1.0859018439499026e-06
   DEBUG:root:i=738 residual=1.0858910476428498e-06
   DEBUG:root:i=739 residual=1.0858802206051113e-06
   DEBUG:root:i=740 residual=1.0858694366117568e-06
   DEBUG:root:i=741 residual=1.0858585603706981e-06
   DEBUG:root:i=742 residual=1.0858477332895903e-06
   DEBUG:root:i=743 residual=1.0858369246911894e-06
   DEBUG:root:i=744 residual=1.085826073033248e-06
   DEBUG:root:i=745 residual=1.0858152398209329e-06
   DEBUG:root:i=746 residual=1.0858043820036978e-06
   DEBUG:root:i=747 residual=1.085793579541527e-06
   DEBUG:root:i=748 residual=1.0857827340458549e-06
   DEBUG:root:i=749 residual=1.0857719623454653e-06
   DEBUG:root:i=750 residual=1.0857611168651838e-06
   DEBUG:root:i=751 residual=1.085750289805299e-06
   DEBUG:root:i=752 residual=1.0857394812048437e-06
   DEBUG:root:i=753 residual=1.085728611096745e-06
   DEBUG:root:i=754 residual=1.0857178209387756e-06
   DEBUG:root:i=755 residual=1.0857070000473998e-06
   DEBUG:root:i=756 residual=1.085696179151468e-06
   DEBUG:root:i=757 residual=1.0856853582527371e-06
   DEBUG:root:i=758 residual=1.0856745435056811e-06
   DEBUG:root:i=759 residual=1.0856636980027118e-06
   DEBUG:root:i=760 residual=1.085652877093668e-06
   DEBUG:root:i=761 residual=1.0856420438939585e-06
   DEBUG:root:i=762 residual=1.0856312352851557e-06
   DEBUG:root:i=763 residual=1.0856204390013844e-06
   DEBUG:root:i=764 residual=1.0856095996631732e-06
   DEBUG:root:i=765 residual=1.085598778756813e-06
   DEBUG:root:i=766 residual=1.0855879640045149e-06
   DEBUG:root:i=767 residual=1.0855771492627763e-06
   DEBUG:root:i=768 residual=1.0855663037648041e-06
   DEBUG:root:i=769 residual=1.0855554705441927e-06
   DEBUG:root:i=770 residual=1.0855446434888302e-06
   DEBUG:root:i=771 residual=1.0855338410449142e-06
   DEBUG:root:i=772 residual=1.0855230263027682e-06
   DEBUG:root:i=773 residual=1.0855121808054684e-06
   DEBUG:root:i=774 residual=1.0855013598908557e-06
   DEBUG:root:i=775 residual=1.0854905512925664e-06
   DEBUG:root:i=776 residual=1.085479736556495e-06
   DEBUG:root:i=777 residual=1.0854689095059379e-06
   DEBUG:root:i=778 residual=1.0854580824580335e-06
   DEBUG:root:i=779 residual=1.0854472677034967e-06
   DEBUG:root:i=780 residual=1.0854364345060758e-06
   DEBUG:root:i=781 residual=1.085425588995399e-06
   DEBUG:root:i=782 residual=1.0854147557801375e-06
   DEBUG:root:i=783 residual=1.0854039533282577e-06
   DEBUG:root:i=784 residual=1.08539314473799e-06
   DEBUG:root:i=785 residual=1.0853823361560873e-06
   DEBUG:root:i=786 residual=1.0853715337177386e-06
   DEBUG:root:i=787 residual=1.0853606943716429e-06
   DEBUG:root:i=788 residual=1.0853498488616021e-06
   DEBUG:root:i=789 residual=1.0853390587054425e-06
   DEBUG:root:i=790 residual=1.0853282501227565e-06
   DEBUG:root:i=791 residual=1.0853173984787905e-06
   DEBUG:root:i=792 residual=1.0853065468035512e-06
   DEBUG:root:i=793 residual=1.0852957443438507e-06
   DEBUG:root:i=794 residual=1.0852849480594766e-06
   DEBUG:root:i=795 residual=1.0852741148677661e-06
   DEBUG:root:i=796 residual=1.0852633062726407e-06
   DEBUG:root:i=797 residual=1.0852524915285244e-06
   DEBUG:root:i=798 residual=1.0852416706352387e-06
   DEBUG:root:i=799 residual=1.0852308866465064e-06
   DEBUG:root:i=800 residual=1.0852200288587178e-06
   DEBUG:root:i=801 residual=1.085209214093855e-06
   DEBUG:root:i=802 residual=1.0851983685851236e-06
   DEBUG:root:i=803 residual=1.0851875661311048e-06
   DEBUG:root:i=804 residual=1.0851767698519569e-06
   DEBUG:root:i=805 residual=1.0851659305088212e-06
   DEBUG:root:i=806 residual=1.0851550972989769e-06
   DEBUG:root:i=807 residual=1.0851443009994347e-06
   DEBUG:root:i=808 residual=1.08513346780984e-06
   DEBUG:root:i=809 residual=1.0851226592096886e-06
   DEBUG:root:i=810 residual=1.0851118260181173e-06
   DEBUG:root:i=811 residual=1.0851010112605467e-06
   DEBUG:root:i=812 residual=1.085090184209781e-06
   DEBUG:root:i=813 residual=1.0850793817692902e-06
   DEBUG:root:i=814 residual=1.0850685731819772e-06
   DEBUG:root:i=815 residual=1.0850577522881355e-06
   DEBUG:root:i=816 residual=1.0850469190888976e-06
   DEBUG:root:i=817 residual=1.085036110485748e-06
   DEBUG:root:i=818 residual=1.085025289594944e-06
   DEBUG:root:i=819 residual=1.0850144686936244e-06
   DEBUG:root:i=820 residual=1.0850036047352946e-06
   DEBUG:root:i=821 residual=1.0849928207286173e-06
   DEBUG:root:i=822 residual=1.08498202444736e-06
   DEBUG:root:i=823 residual=1.0849712158698305e-06
   DEBUG:root:i=824 residual=1.084960376521767e-06
   DEBUG:root:i=825 residual=1.0849495433148676e-06
   DEBUG:root:i=826 residual=1.0849387285572076e-06
   DEBUG:root:i=827 residual=1.0849279015117818e-06
   DEBUG:root:i=828 residual=1.0849171175183784e-06
   DEBUG:root:i=829 residual=1.0849062904883667e-06
   DEBUG:root:i=830 residual=1.0848954634383962e-06
   DEBUG:root:i=831 residual=1.0848846548354246e-06
   DEBUG:root:i=832 residual=1.0848738339468556e-06
   DEBUG:root:i=833 residual=1.0848630315011452e-06
   DEBUG:root:i=834 residual=1.0848521798568471e-06
   DEBUG:root:i=835 residual=1.0848414019992898e-06
   DEBUG:root:i=836 residual=1.084830624191111e-06
   DEBUG:root:i=837 residual=1.084819791007986e-06
   DEBUG:root:i=838 residual=1.084808982412824e-06
   DEBUG:root:i=839 residual=1.0847981246066513e-06
   DEBUG:root:i=840 residual=1.0847873406001563e-06
   DEBUG:root:i=841 residual=1.0847765443288629e-06
   DEBUG:root:i=842 residual=1.0847656988315401e-06
   DEBUG:root:i=843 residual=1.084754920989622e-06
   DEBUG:root:i=844 residual=1.084744069350681e-06
   DEBUG:root:i=845 residual=1.08473326074003e-06
   DEBUG:root:i=846 residual=1.0847224336970708e-06
   DEBUG:root:i=847 residual=1.0847116374054752e-06
   DEBUG:root:i=848 residual=1.0847008226664004e-06
   DEBUG:root:i=849 residual=1.0846900202308599e-06
   DEBUG:root:i=850 residual=1.0846791808850713e-06
   DEBUG:root:i=851 residual=1.0846684091923979e-06
   DEBUG:root:i=852 residual=1.0846575637100547e-06
   DEBUG:root:i=853 residual=1.0846467612592815e-06
   DEBUG:root:i=854 residual=1.0846359649727174e-06
   DEBUG:root:i=855 residual=1.084625131786157e-06
   DEBUG:root:i=856 residual=1.084614323186088e-06
   DEBUG:root:i=857 residual=1.0846034838372906e-06
   DEBUG:root:i=858 residual=1.0845926936922576e-06
   DEBUG:root:i=859 residual=1.0845818481967733e-06
   DEBUG:root:i=860 residual=1.0845710457430097e-06
   DEBUG:root:i=861 residual=1.0845602248544145e-06
   DEBUG:root:i=862 residual=1.0845494101079313e-06
   DEBUG:root:i=863 residual=1.0845386384253225e-06
   DEBUG:root:i=864 residual=1.0845277867944089e-06
   DEBUG:root:i=865 residual=1.084516996639296e-06
   DEBUG:root:i=866 residual=1.0845061634524951e-06
   DEBUG:root:i=867 residual=1.0844953610037743e-06
   DEBUG:root:i=868 residual=1.0844845647225476e-06
   DEBUG:root:i=869 residual=1.0844737315365158e-06
   DEBUG:root:i=870 residual=1.0844629352365507e-06
   DEBUG:root:i=871 residual=1.0844521205004062e-06
   DEBUG:root:i=872 residual=1.0844412934580281e-06
   DEBUG:root:i=873 residual=1.0844305033155852e-06
   DEBUG:root:i=874 residual=1.0844197008847135e-06
   DEBUG:root:i=875 residual=1.084408898454813e-06
   DEBUG:root:i=876 residual=1.084398077564139e-06
   DEBUG:root:i=877 residual=1.0843872505168195e-06
   DEBUG:root:i=878 residual=1.0843764788292147e-06
   DEBUG:root:i=879 residual=1.0843656333469757e-06
   DEBUG:root:i=880 residual=1.0843548062864982e-06
   DEBUG:root:i=881 residual=1.0843440161465908e-06
   DEBUG:root:i=882 residual=1.084333201412435e-06
   DEBUG:root:i=883 residual=1.0843223928171817e-06
   DEBUG:root:i=884 residual=1.0843115903875862e-06
   DEBUG:root:i=885 residual=1.0843008002525063e-06
   DEBUG:root:i=886 residual=1.0842899916806633e-06
   DEBUG:root:i=887 residual=1.0842791461786014e-06
   DEBUG:root:i=888 residual=1.0842683314159003e-06
   DEBUG:root:i=889 residual=1.0842575166789835e-06
   DEBUG:root:i=890 residual=1.084246751146032e-06
   DEBUG:root:i=891 residual=1.084235930276231e-06
   DEBUG:root:i=892 residual=1.0842250847736946e-06
   DEBUG:root:i=893 residual=1.084214282317385e-06
   DEBUG:root:i=894 residual=1.0842034737297175e-06
   DEBUG:root:i=895 residual=1.0841926712965783e-06
   DEBUG:root:i=896 residual=1.0841818934707631e-06
   DEBUG:root:i=897 residual=1.0841710664363145e-06
   DEBUG:root:i=898 residual=1.0841602763017765e-06
   DEBUG:root:i=899 residual=1.0841494308092308e-06
   DEBUG:root:i=900 residual=1.0841386160525658e-06
   DEBUG:root:i=901 residual=1.0841278074619429e-06
   DEBUG:root:i=902 residual=1.084116998867035e-06
   DEBUG:root:i=903 residual=1.0841062148917025e-06
   DEBUG:root:i=904 residual=1.0840953817080046e-06
   DEBUG:root:i=905 residual=1.084084573107839e-06
   DEBUG:root:i=906 residual=1.0840737645178866e-06
   DEBUG:root:i=907 residual=1.08406292517177e-06
   DEBUG:root:i=908 residual=1.0840521719348285e-06
   DEBUG:root:i=909 residual=1.0840413756767232e-06
   DEBUG:root:i=910 residual=1.0840305670976023e-06
   DEBUG:root:i=911 residual=1.084019776960732e-06
   DEBUG:root:i=912 residual=1.0840089376279742e-06
   DEBUG:root:i=913 residual=1.0839981536293564e-06
   DEBUG:root:i=914 residual=1.0839873327492425e-06
   DEBUG:root:i=915 residual=1.0839765303112235e-06
   DEBUG:root:i=916 residual=1.083965697116966e-06
   DEBUG:root:i=917 residual=1.0839549008176439e-06
   DEBUG:root:i=918 residual=1.083944116844598e-06
   DEBUG:root:i=919 residual=1.0839333082680603e-06
   DEBUG:root:i=920 residual=1.083922493529369e-06
   DEBUG:root:i=921 residual=1.0839116849395456e-06
   DEBUG:root:i=922 residual=1.083900870197765e-06
   DEBUG:root:i=923 residual=1.0838900923667536e-06
   DEBUG:root:i=924 residual=1.0838792468824114e-06
   DEBUG:root:i=925 residual=1.0838684505801884e-06
   DEBUG:root:i=926 residual=1.0838576296922206e-06
   DEBUG:root:i=927 residual=1.0838468026468035e-06
   DEBUG:root:i=928 residual=1.0838360063499657e-06
   DEBUG:root:i=929 residual=1.0838251916163119e-06
   DEBUG:root:i=930 residual=1.0838144076303663e-06
   DEBUG:root:i=931 residual=1.0838035990563422e-06
   DEBUG:root:i=932 residual=1.0837928089246155e-06
   DEBUG:root:i=933 residual=1.0837819818897915e-06
   DEBUG:root:i=934 residual=1.0837711855987272e-06
   DEBUG:root:i=935 residual=1.0837603585587397e-06
   DEBUG:root:i=936 residual=1.0837495561102734e-06
   DEBUG:root:i=937 residual=1.0837387659856243e-06
   DEBUG:root:i=938 residual=1.0837279758616145e-06
   DEBUG:root:i=939 residual=1.0837172103445343e-06
   DEBUG:root:i=940 residual=1.0837063894779016e-06
   DEBUG:root:i=941 residual=1.083695568576996e-06
   DEBUG:root:i=942 residual=1.0836847722880718e-06
   DEBUG:root:i=943 residual=1.0836739083429443e-06
   DEBUG:root:i=944 residual=1.0836631304829297e-06
   DEBUG:root:i=945 residual=1.0836523280626453e-06
   DEBUG:root:i=946 residual=1.0836415379284503e-06
   DEBUG:root:i=947 residual=1.0836307416600548e-06
   DEBUG:root:i=948 residual=1.0836199269239882e-06
   DEBUG:root:i=949 residual=1.0836091306378742e-06
   DEBUG:root:i=950 residual=1.083598309749889e-06
   DEBUG:root:i=951 residual=1.0835875196154493e-06
   DEBUG:root:i=952 residual=1.0835767294883948e-06
   DEBUG:root:i=953 residual=1.0835659332152174e-06
   DEBUG:root:i=954 residual=1.0835550815715197e-06
   DEBUG:root:i=955 residual=1.0835442975699695e-06
   DEBUG:root:i=956 residual=1.0835334951426094e-06
   DEBUG:root:i=957 residual=1.0835226558018666e-06
   DEBUG:root:i=958 residual=1.0835119087161067e-06
   DEBUG:root:i=959 residual=1.0835010940056125e-06
   DEBUG:root:i=960 residual=1.0834902731103399e-06
   DEBUG:root:i=961 residual=1.0834794829758378e-06
   DEBUG:root:i=962 residual=1.0834686620927396e-06
   DEBUG:root:i=963 residual=1.0834578473437007e-06
   DEBUG:root:i=964 residual=1.0834470449053836e-06
   DEBUG:root:i=965 residual=1.0834362732312983e-06
   DEBUG:root:i=966 residual=1.0834254646619487e-06
   DEBUG:root:i=967 residual=1.0834147052892117e-06
   DEBUG:root:i=968 residual=1.0834038905765694e-06
   DEBUG:root:i=969 residual=1.0833930819897096e-06
   DEBUG:root:i=970 residual=1.0833822610912352e-06
   DEBUG:root:i=971 residual=1.0833714648024328e-06
   DEBUG:root:i=972 residual=1.0833606685263544e-06
   DEBUG:root:i=973 residual=1.0833498783970591e-06
   DEBUG:root:i=974 residual=1.0833390698228148e-06
   DEBUG:root:i=975 residual=1.083328255081706e-06
   DEBUG:root:i=976 residual=1.0833174649471032e-06
   DEBUG:root:i=977 residual=1.0833066686684866e-06
   DEBUG:root:i=978 residual=1.0832958416366955e-06
   DEBUG:root:i=979 residual=1.0832850576436751e-06
   DEBUG:root:i=980 residual=1.0832742490690331e-06
   DEBUG:root:i=981 residual=1.0832634835418066e-06
   DEBUG:root:i=982 residual=1.0832526626721413e-06
   DEBUG:root:i=983 residual=1.0832418602366552e-06
   DEBUG:root:i=984 residual=1.0832310639529955e-06
   DEBUG:root:i=985 residual=1.0832202492172133e-06
   DEBUG:root:i=986 residual=1.0832094467762862e-06
   DEBUG:root:i=987 residual=1.0831986566519848e-06
   DEBUG:root:i=988 residual=1.0831878849803845e-06
   DEBUG:root:i=989 residual=1.0831770456558978e-06
   DEBUG:root:i=990 residual=1.08316630472253e-06
   DEBUG:root:i=991 residual=1.083155471556384e-06
   DEBUG:root:i=992 residual=1.0831446629623263e-06
   DEBUG:root:i=993 residual=1.0831339035889347e-06
   DEBUG:root:i=994 residual=1.0831230888733508e-06
   DEBUG:root:i=995 residual=1.0831123171916865e-06
   DEBUG:root:i=996 residual=1.0831014901729762e-06
   DEBUG:root:i=997 residual=1.0830906815708327e-06
   DEBUG:root:i=998 residual=1.0830798975925257e-06
   DEBUG:root:i=999 residual=1.0830690890186466e-06
   INFO:root:rank=0 pagerank=1.0716e+01 url=www.lawfareblog.com/lawfare-job-board
   INFO:root:rank=1 pagerank=1.0716e+01 url=www.lawfareblog.com/snowden-revelations
   INFO:root:rank=2 pagerank=1.0716e+01 url=www.lawfareblog.com/support-lawfare
   INFO:root:rank=3 pagerank=1.0716e+01 url=www.lawfareblog.com/upcoming-events
   INFO:root:rank=4 pagerank=1.0716e+01 url=www.lawfareblog.com/our-comments-policy
   INFO:root:rank=5 pagerank=1.0716e+01 url=www.lawfareblog.com/subscribe-lawfare
   INFO:root:rank=6 pagerank=1.0716e+01 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban
   INFO:root:rank=7 pagerank=1.0716e+01 url=www.lawfareblog.com/cyberlaw-podcast-sandworm-and-grus-global-intifada
   INFO:root:rank=8 pagerank=1.0716e+01 url=www.lawfareblog.com/cyberlaw-podcast-plumbing-depths-artificial-stupidity
   INFO:root:rank=9 pagerank=1.0716e+01 url=www.lawfareblog.com/doj-charges-two-former-twitter-employees-spying-saudis




   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2
   INFO:gensim.models.keyedvectors:loading projection weights from /home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz
   DEBUG:smart_open.smart_open_lib:{'transport_params': None, 'compression': 'infer_from_extension', 'opener': None, 'closefd': True, 'newline': None, 'errors': None, 'encoding': None, 'buffering': -1, 'mode': 'rb', 'uri': '/home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz'}
   DEBUG:gensim.utils:starting a new internal lifecycle event log for KeyedVectors
   INFO:gensim.utils:KeyedVectors lifecycle event {'msg': 'loaded (3000000, 300) matrix of type float32 from /home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz', 'binary': True, 'encoding': 'utf8', 'datetime': '2022-12-11T20:49:37.166200', 'gensim': '4.2.0', 'python': '3.6.9 (default, Dec  8 2021, 21:08:43) \n[GCC 8.4.0]', 'platform': 'Linux-4.15.0-88-generic-x86_64-with-Ubuntu-18.04-bionic', 'event': 'load_word2vec_format'}
   DEBUG:root:computing indices
   DEBUG:root:computing values
   DEBUG:root:i=0 residual=5.117074346273421
   DEBUG:root:i=1 residual=2.980514885934993
   DEBUG:root:i=2 residual=2.4492398105409228
   DEBUG:root:i=3 residual=1.6986576427801796
   DEBUG:root:i=4 residual=1.1001056307665857
   DEBUG:root:i=5 residual=0.7297587235496557
   DEBUG:root:i=6 residual=0.5147641455152537
   DEBUG:root:i=7 residual=0.379582105825058
   DEBUG:root:i=8 residual=0.28508515837926746
   DEBUG:root:i=9 residual=0.21570594495113743
   DEBUG:root:i=10 residual=0.1648312084595632
   DEBUG:root:i=11 residual=0.12841500310203333
   DEBUG:root:i=12 residual=0.1030473644123785
   DEBUG:root:i=13 residual=0.08559392282150778
   DEBUG:root:i=14 residual=0.07334336292050049
   DEBUG:root:i=15 residual=0.0642308410861129
   DEBUG:root:i=16 residual=0.0568982784559337
   DEBUG:root:i=17 residual=0.05057048706778108
   DEBUG:root:i=18 residual=0.04486214335858702
   DEBUG:root:i=19 residual=0.039610586844404085
   DEBUG:root:i=20 residual=0.034763556633105444
   DEBUG:root:i=21 residual=0.03031425116639416
   DEBUG:root:i=22 residual=0.026267806098506877
   DEBUG:root:i=23 residual=0.022626128757745508
   DEBUG:root:i=24 residual=0.019382585872490407
   DEBUG:root:i=25 residual=0.016521549658255058
   DEBUG:root:i=26 residual=0.014020041008978463
   DEBUG:root:i=27 residual=0.011850022629751465
   DEBUG:root:i=28 residual=0.009980636486228244
   DEBUG:root:i=29 residual=0.008380085110775393
   DEBUG:root:i=30 residual=0.007017069059321187
   DEBUG:root:i=31 residual=0.005861797412834496
   DEBUG:root:i=32 residual=0.004886633256459956
   DEBUG:root:i=33 residual=0.00406644939020625
   DEBUG:root:i=34 residual=0.003378766955020687
   DEBUG:root:i=35 residual=0.0028037400540904586
   DEBUG:root:i=36 residual=0.002324037618958225
   DEBUG:root:i=37 residual=0.0019246621802931011
   DEBUG:root:i=38 residual=0.0015927349858779972
   DEBUG:root:i=39 residual=0.0013172684561064561
   DEBUG:root:i=40 residual=0.0010889402835975765
   DEBUG:root:i=41 residual=0.0008998783863254429
   DEBUG:root:i=42 residual=0.0007434621604770279
   DEBUG:root:i=43 residual=0.0006141427889412211
   DEBUG:root:i=44 residual=0.0005072835049861228
   DEBUG:root:i=45 residual=0.00041901948411883895
   DEBUG:root:i=46 residual=0.00034613627366900644
   DEBUG:root:i=47 residual=0.00028596523773344334
   DEBUG:root:i=48 residual=0.00023629429382363636
   DEBUG:root:i=49 residual=0.00019529217109375246
   DEBUG:root:i=50 residual=0.0001614444729081887
   DEBUG:root:i=51 residual=0.0001334999390125067
   DEBUG:root:i=52 residual=0.00011042544681616451
   DEBUG:root:i=53 residual=9.136844850387412e-05
   DEBUG:root:i=54 residual=7.562569841209722e-05
   DEBUG:root:i=55 residual=6.261727575986882e-05
   DEBUG:root:i=56 residual=5.1865046897101944e-05
   DEBUG:root:i=57 residual=4.297483670708253e-05
   DEBUG:root:i=58 residual=3.562168985062296e-05
   DEBUG:root:i=59 residual=2.953769961365253e-05
   DEBUG:root:i=60 residual=2.4501965917664844e-05
   DEBUG:root:i=61 residual=2.0332315853379843e-05
   DEBUG:root:i=62 residual=1.6878481097552326e-05
   DEBUG:root:i=63 residual=1.4016478126683178e-05
   DEBUG:root:i=64 residual=1.164398048612946e-05
   DEBUG:root:i=65 residual=9.67650864359728e-06
   DEBUG:root:i=66 residual=8.04429323830644e-06
   DEBUG:root:i=67 residual=6.689692690339965e-06
   DEBUG:root:i=68 residual=5.565067042441542e-06
   DEBUG:root:i=69 residual=4.631027177878151e-06
   DEBUG:root:i=70 residual=3.854992868180266e-06
   DEBUG:root:i=71 residual=3.210004877836229e-06
   DEBUG:root:i=72 residual=2.6737460947919805e-06
   DEBUG:root:i=73 residual=2.2277346478224032e-06
   DEBUG:root:i=74 residual=1.8566585630777452e-06
   DEBUG:root:i=75 residual=1.5478269278153886e-06
   DEBUG:root:i=76 residual=1.2907169894424181e-06
   DEBUG:root:i=77 residual=1.0766002649527334e-06
   DEBUG:root:i=78 residual=8.98233752542629e-07
   INFO:root:rank=0 pagerank=4.6091e+00 url=www.lawfareblog.com/0-days-n-days-iphones-and-android
   INFO:root:rank=1 pagerank=2.9867e+00 url=www.lawfareblog.com/lawfare-job-board
   INFO:root:rank=2 pagerank=2.9669e+00 url=www.lawfareblog.com/doj-charges-two-former-twitter-employees-spying-saudis
   INFO:root:rank=3 pagerank=2.0173e+00 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban
   INFO:root:rank=4 pagerank=1.8769e+00 url=www.lawfareblog.com/subscribe-lawfare
   INFO:root:rank=5 pagerank=1.8762e+00 url=www.lawfareblog.com/lessons-so-far-whatsapp-v-nso
   INFO:root:rank=6 pagerank=1.8693e+00 url=www.lawfareblog.com/cyberlaw-podcast-plumbing-depths-artificial-stupidity
   INFO:root:rank=7 pagerank=1.7655e+00 url=www.lawfareblog.com/our-comments-policy
   INFO:root:rank=8 pagerank=1.6807e+00 url=www.lawfareblog.com/upcoming-events
   INFO:root:rank=9 pagerank=9.8345e-01 url=www.lawfareblog.com/cyberlaw-podcast-sandworm-and-grus-global-intifada





   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2 --alpha=0.99999
   INFO:gensim.models.keyedvectors:loading projection weights from /home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz
   DEBUG:smart_open.smart_open_lib:{'transport_params': None, 'compression': 'infer_from_extension', 'opener': None, 'closefd': True, 'newline': None, 'errors': None, 'encoding': None, 'buffering': -1, 'mode': 'rb', 'uri': '/home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz'}
   DEBUG:gensim.utils:starting a new internal lifecycle event log for KeyedVectors
   INFO:gensim.utils:KeyedVectors lifecycle event {'msg': 'loaded (3000000, 300) matrix of type float32 from /home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz', 'binary': True, 'encoding': 'utf8', 'datetime': '2022-12-11T20:51:43.411599', 'gensim': '4.2.0', 'python': '3.6.9 (default, Dec  8 2021, 21:08:43) \n[GCC 8.4.0]', 'platform': 'Linux-4.15.0-88-generic-x86_64-with-Ubuntu-18.04-bionic', 'event': 'load_word2vec_format'}
   DEBUG:root:computing indices
   DEBUG:root:computing values
   DEBUG:root:i=0 residual=6.020027026830456
   DEBUG:root:i=1 residual=4.12520132307857
   DEBUG:root:i=2 residual=3.9880960156988863
   DEBUG:root:i=3 residual=3.2540411541820866
   DEBUG:root:i=4 residual=2.4793362066620466
   DEBUG:root:i=5 residual=1.9349151716649848
   DEBUG:root:i=6 residual=1.605713069323117
   DEBUG:root:i=7 residual=1.3929615412309546
   DEBUG:root:i=8 residual=1.2307796864257625
   DEBUG:root:i=9 residual=1.0955676425159209
   DEBUG:root:i=10 residual=0.9848879369268593
   DEBUG:root:i=11 residual=0.9026775603365826
   DEBUG:root:i=12 residual=0.8521650632427237
   DEBUG:root:i=13 residual=0.8327256015785429
   DEBUG:root:i=14 residual=0.8394507083664201
   DEBUG:root:i=15 residual=0.8648811385702684
   DEBUG:root:i=16 residual=0.9013479847552873
   DEBUG:root:i=17 residual=0.9424801910168211
   DEBUG:root:i=18 residual=0.9836427452172722
   DEBUG:root:i=19 residual=1.0217652876957557
   DEBUG:root:i=20 residual=1.0549858361429165
   DEBUG:root:i=21 residual=1.0823104495674953
   DEBUG:root:i=22 residual=1.1033452169311453
   DEBUG:root:i=23 residual=1.1180997041768412
   DEBUG:root:i=24 residual=1.1268464720390967
   DEBUG:root:i=25 residual=1.130021466082933
   DEBUG:root:i=26 residual=1.1281536125107094
   DEBUG:root:i=27 residual=1.1218153173127525
   DEBUG:root:i=28 residual=1.111588058067756
   DEBUG:root:i=29 residual=1.0980389676992284
   DEBUG:root:i=30 residual=1.081705464356969
   DEBUG:root:i=31 residual=1.0630857710280184
   DEBUG:root:i=32 residual=1.042633721818853
   DEBUG:root:i=33 residual=1.0207566509229098
   DEBUG:root:i=34 residual=0.9978154556467971
   DEBUG:root:i=35 residual=0.9741261478996461
   DEBUG:root:i=36 residual=0.9499623792343981
   DEBUG:root:i=37 residual=0.9255585561038899
   DEBUG:root:i=38 residual=0.9011132636392937
   DEBUG:root:i=39 residual=0.8767927946389674
   DEBUG:root:i=40 residual=0.8527346405957138
   DEBUG:root:i=41 residual=0.8290508473845242
   DEBUG:root:i=42 residual=0.8058311727532997
   DEBUG:root:i=43 residual=0.7831460084515753
   DEBUG:root:i=44 residual=0.7610490486514608
   DEBUG:root:i=45 residual=0.7395796998211449
   DEBUG:root:i=46 residual=0.7187652366447439
   DEBUG:root:i=47 residual=0.698622714926127
   DEBUG:root:i=48 residual=0.6791606564395805
   DEBUG:root:i=49 residual=0.6603805230021028
   DEBUG:root:i=50 residual=0.6422779981059293
   DEBUG:root:i=51 residual=0.6248440946267918
   DEBUG:root:i=52 residual=0.6080661066857553
   DEBUG:root:i=53 residual=0.5919284228972094
   DEBUG:root:i=54 residual=0.5764132171359085
   DEBUG:root:i=55 residual=0.5615010317147863
   DEBUG:root:i=56 residual=0.5471712665646147
   DEBUG:root:i=57 residual=0.5334025867029658
   DEBUG:root:i=58 residual=0.5201732590140475
   DEBUG:root:i=59 residual=0.5074614281578733
   DEBUG:root:i=60 residual=0.4952453403030999
   DEBUG:root:i=61 residual=0.48350352234136484
   DEBUG:root:i=62 residual=0.47221492329520537
   DEBUG:root:i=63 residual=0.46135902377639204
   DEBUG:root:i=64 residual=0.4509159185835875
   DEBUG:root:i=65 residual=0.4408663768435937
   DEBUG:root:i=66 residual=0.4311918834927737
   DEBUG:root:i=67 residual=0.4218746653593348
   DEBUG:root:i=68 residual=0.4128977046359917
   DEBUG:root:i=69 residual=0.4042447421204449
   DEBUG:root:i=70 residual=0.3959002722418216
   DEBUG:root:i=71 residual=0.38784953157935204
   DEBUG:root:i=72 residual=0.38007848230937524
   DEBUG:root:i=73 residual=0.3725737917839586
   DEBUG:root:i=74 residual=0.3653228092441622
   DEBUG:root:i=75 residual=0.3583135404991701
   DEBUG:root:i=76 residual=0.3515346212562672
   DEBUG:root:i=77 residual=0.34497528966144164
   DEBUG:root:i=78 residual=0.3386253585046731
   DEBUG:root:i=79 residual=0.3324751874547719
   DEBUG:root:i=80 residual=0.3265156556123136
   DEBUG:root:i=81 residual=0.3207381346070809
   DEBUG:root:i=82 residual=0.3151344624127451
   DEBUG:root:i=83 residual=0.3096969180082251
   DEBUG:root:i=84 residual=0.30441819697808553
   DEBUG:root:i=85 residual=0.29929138811547235
   DEBUG:root:i=86 residual=0.2943099510659009
   DEBUG:root:i=87 residual=0.2894676950308794
   DEBUG:root:i=88 residual=0.28475875853463367
   DEBUG:root:i=89 residual=0.2801775902443156
   DEBUG:root:i=90 residual=0.27571893082455673
   DEBUG:root:i=91 residual=0.2713777957998248
   DEBUG:root:i=92 residual=0.2671494593918265
   DEBUG:root:i=93 residual=0.26302943929622535
   DEBUG:root:i=94 residual=0.25901348235890714
   DEBUG:root:i=95 residual=0.25509755111125737
   DEBUG:root:i=96 residual=0.251277811122646
   DEBUG:root:i=97 residual=0.24755061912810158
   DEBUG:root:i=98 residual=0.24391251188933918
   DEBUG:root:i=99 residual=0.2403601957487513
   DEBUG:root:i=100 residual=0.23689053683575476
   DEBUG:root:i=101 residual=0.23350055188750285
   DEBUG:root:i=102 residual=0.23018739964654555
   DEBUG:root:i=103 residual=0.2269483727999967
   DEBUG:root:i=104 residual=0.22378089042618737
   DEBUG:root:i=105 residual=0.22068249091692826
   DEBUG:root:i=106 residual=0.2176508253442933
   DEBUG:root:i=107 residual=0.2146836512437686
   DEBUG:root:i=108 residual=0.21177882678623103
   DEBUG:root:i=109 residual=0.20893430531307347
   DEBUG:root:i=110 residual=0.20614813021086634
   DEBUG:root:i=111 residual=0.20341843010292238
   DEBUG:root:i=112 residual=0.20074341433641646
   DEBUG:root:i=113 residual=0.1981213687458343
   DEBUG:root:i=114 residual=0.19555065167399838
   DEBUG:root:i=115 residual=0.19302969023363414
   DEBUG:root:i=116 residual=0.19055697679353376
   DEBUG:root:i=117 residual=0.18813106567406793
   DEBUG:root:i=118 residual=0.18575057003867435
   DEBUG:root:i=119 residual=0.18341415896782348
   DEBUG:root:i=120 residual=0.18112055470362162
   DEBUG:root:i=121 residual=0.17886853005409958
   DEBUG:root:i=122 residual=0.1766569059462089
   DEBUG:root:i=123 residual=0.17448454911830213
   DEBUG:root:i=124 residual=0.17235036994279257
   DEBUG:root:i=125 residual=0.1702533203707016
   DEBUG:root:i=126 residual=0.16819239199017433
   DEBUG:root:i=127 residual=0.1661666141920132
   DEBUG:root:i=128 residual=0.16417505243489022
   DEBUG:root:i=129 residual=0.16221680660464796
   DEBUG:root:i=130 residual=0.16029100946123268
   DEBUG:root:i=131 residual=0.1583968251682095
   DEBUG:root:i=132 residual=0.1565334478996073
   DEBUG:root:i=133 residual=0.15470010051945038
   DEBUG:root:i=134 residual=0.15289603332941576
   DEBUG:root:i=135 residual=0.15112052288092936
   DEBUG:root:i=136 residual=0.14937287084727888
   DEBUG:root:i=137 residual=0.14765240295268228
   DEBUG:root:i=138 residual=0.14595846795483886
   DEBUG:root:i=139 residual=0.14429043667771702
   DEBUG:root:i=140 residual=0.14264770109188007
   DEBUG:root:i=141 residual=0.14102967343948342
   DEBUG:root:i=142 residual=0.1394357854016269
   DEBUG:root:i=143 residual=0.13786548730544476
   DEBUG:root:i=144 residual=0.13631824736884954
   DEBUG:root:i=145 residual=0.13479355098090987
   DEBUG:root:i=146 residual=0.13329090001578253
   DEBUG:root:i=147 residual=0.1318098121786137
   DEBUG:root:i=148 residual=0.1303498203812784
   DEBUG:root:i=149 residual=0.128910472146944
   DEBUG:root:i=150 residual=0.12749132904137117
   DEBUG:root:i=151 residual=0.12609196613001955
   DEBUG:root:i=152 residual=0.12471197145928531
   DEBUG:root:i=153 residual=0.12335094556093218
   DEBUG:root:i=154 residual=0.12200850097819756
   DEBUG:root:i=155 residual=0.1206842618127123
   DEBUG:root:i=156 residual=0.11937786329125875
   DEBUG:root:i=157 residual=0.11808895135110192
   DEBUG:root:i=158 residual=0.11681718224328817
   DEBUG:root:i=159 residual=0.11556222215268888
   DEBUG:root:i=160 residual=0.11432374683444665
   DEBUG:root:i=161 residual=0.1131014412654539
   DEBUG:root:i=162 residual=0.11189499931074581
   DEBUG:root:i=163 residual=0.11070412340360494
   DEBUG:root:i=164 residual=0.10952852423902387
   DEBUG:root:i=165 residual=0.10836792047978047
   DEBUG:root:i=166 residual=0.10722203847464111
   DEBUG:root:i=167 residual=0.10609061198792766
   DEBUG:root:i=168 residual=0.10497338194023144
   DEBUG:root:i=169 residual=0.10387009615950585
   DEBUG:root:i=170 residual=0.10278050914210712
   DEBUG:root:i=171 residual=0.1017043818235933
   DEBUG:root:i=172 residual=0.10064148135829011
   DEBUG:root:i=173 residual=0.09959158090814477
   DEBUG:root:i=174 residual=0.09855445943919437
   DEBUG:root:i=175 residual=0.09752990152662995
   DEBUG:root:i=176 residual=0.09651769716715076
   DEBUG:root:i=177 residual=0.09551764159872692
   DEBUG:root:i=178 residual=0.09452953512721746
   DEBUG:root:i=179 residual=0.09355318295983764
   DEBUG:root:i=180 residual=0.09258839504490672
   DEBUG:root:i=181 residual=0.09163498591765785
   DEBUG:root:i=182 residual=0.09069277455210896
   DEBUG:root:i=183 residual=0.08976158421828223
   DEBUG:root:i=184 residual=0.08884124234504723
   DEBUG:root:i=185 residual=0.087931580387901
   DEBUG:root:i=186 residual=0.08703243370186656
   DEBUG:root:i=187 residual=0.08614364141899951
   DEBUG:root:i=188 residual=0.08526504633049033
   DEBUG:root:i=189 residual=0.08439649477310575
   DEBUG:root:i=190 residual=0.08353783651974943
   DEBUG:root:i=191 residual=0.08268892467421579
   DEBUG:root:i=192 residual=0.08184961556945367
   DEBUG:root:i=193 residual=0.08101976866987136
   DEBUG:root:i=194 residual=0.08019924647691505
   DEBUG:root:i=195 residual=0.07938791443821142
   DEBUG:root:i=196 residual=0.07858564085986611
   DEBUG:root:i=197 residual=0.07779229682193425
   DEBUG:root:i=198 residual=0.07700775609686893
   DEBUG:root:i=199 residual=0.07623189507097133
   DEBUG:root:i=200 residual=0.0754645926683788
   DEBUG:root:i=201 residual=0.07470573027802334
   DEBUG:root:i=202 residual=0.07395519168288076
   DEBUG:root:i=203 residual=0.07321286299179905
   DEBUG:root:i=204 residual=0.07247863257376486
   DEBUG:root:i=205 residual=0.07175239099430228
   DEBUG:root:i=206 residual=0.07103403095405925
   DEBUG:root:i=207 residual=0.07032344722974035
   DEBUG:root:i=208 residual=0.06962053661672309
   DEBUG:root:i=209 residual=0.06892519787379045
   DEBUG:root:i=210 residual=0.0682373316697961
   DEBUG:root:i=211 residual=0.06755684053203256
   DEBUG:root:i=212 residual=0.06688362879626368
   DEBUG:root:i=213 residual=0.0662176025586734
   DEBUG:root:i=214 residual=0.0655586696291198
   DEBUG:root:i=215 residual=0.06490673948607481
   DEBUG:root:i=216 residual=0.064261723233226
   DEBUG:root:i=217 residual=0.06362353355712072
   DEBUG:root:i=218 residual=0.06299208468648834
   DEBUG:root:i=219 residual=0.06236729235284714
   DEBUG:root:i=220 residual=0.06174907375223687
   DEBUG:root:i=221 residual=0.061137347508437224
   DEBUG:root:i=222 residual=0.06053203363706531
   DEBUG:root:i=223 residual=0.05993305351115233
   DEBUG:root:i=224 residual=0.05934032982747933
   DEBUG:root:i=225 residual=0.058753786574286995
   DEBUG:root:i=226 residual=0.05817334899977719
   DEBUG:root:i=227 residual=0.05759894358173971
   DEBUG:root:i=228 residual=0.05703049799802904
   DEBUG:root:i=229 residual=0.05646794109810959
   DEBUG:root:i=230 residual=0.05591120287528249
   DEBUG:root:i=231 residual=0.05536021443990507
   DEBUG:root:i=232 residual=0.0548149079934755
   DEBUG:root:i=233 residual=0.05427521680331493
   DEBUG:root:i=234 residual=0.053741075178203035
   DEBUG:root:i=235 residual=0.053212418444669854
   DEBUG:root:i=236 residual=0.052689182924059566
   DEBUG:root:i=237 residual=0.052171305910156576
   DEBUG:root:i=238 residual=0.05165872564760958
   DEBUG:root:i=239 residual=0.051151381310974124
   DEBUG:root:i=240 residual=0.05064921298427965
   DEBUG:root:i=241 residual=0.05015216164134046
   DEBUG:root:i=242 residual=0.04966016912647547
   DEBUG:root:i=243 residual=0.04917317813596205
   DEBUG:root:i=244 residual=0.04869113219987233
   DEBUG:root:i=245 residual=0.04821397566456891
   DEBUG:root:i=246 residual=0.04774165367553435
   DEBUG:root:i=247 residual=0.047274112160929235
   DEBUG:root:i=248 residual=0.046811297815367234
   DEBUG:root:i=249 residual=0.04635315808425387
   DEBUG:root:i=250 residual=0.045899641148610434
   DEBUG:root:i=251 residual=0.045450695910248175
   DEBUG:root:i=252 residual=0.045006271977421686
   DEBUG:root:i=253 residual=0.04456631965080146
   DEBUG:root:i=254 residual=0.04413078990980408
   DEBUG:root:i=255 residual=0.04369963439944558
   DEBUG:root:i=256 residual=0.04327280541748071
   DEBUG:root:i=257 residual=0.04285025590175498
   DEBUG:root:i=258 residual=0.042431939418067346
   DEBUG:root:i=259 residual=0.04201781014832412
   DEBUG:root:i=260 residual=0.04160782287895625
   DEBUG:root:i=261 residual=0.04120193298964753
   DEBUG:root:i=262 residual=0.04080009644238269
   DEBUG:root:i=263 residual=0.04040226977076213
   DEBUG:root:i=264 residual=0.0400084100696507
   DEBUG:root:i=265 residual=0.03961847498495645
   DEBUG:root:i=266 residual=0.039232422703858655
   DEBUG:root:i=267 residual=0.03885021194509
   DEBUG:root:i=268 residual=0.038471801949590725
   DEBUG:root:i=269 residual=0.038097152471394786
   DEBUG:root:i=270 residual=0.037726223768648245
   DEBUG:root:i=271 residual=0.03735897659496455
   DEBUG:root:i=272 residual=0.03699537219091487
   DEBUG:root:i=273 residual=0.03663537227573931
   DEBUG:root:i=274 residual=0.03627893903930808
   DEBUG:root:i=275 residual=0.035926035134168983
   DEBUG:root:i=276 residual=0.03557662366797288
   DEBUG:root:i=277 residual=0.035230668195790224
   DEBUG:root:i=278 residual=0.03488813271296467
   DEBUG:root:i=279 residual=0.03454898164780033
   DEBUG:root:i=280 residual=0.03421317985471642
   DEBUG:root:i=281 residual=0.033880692607222575
   DEBUG:root:i=282 residual=0.03355148559141767
   DEBUG:root:i=283 residual=0.033225524899392164
   DEBUG:root:i=284 residual=0.032902777022887915
   DEBUG:root:i=285 residual=0.032583208846989675
   DEBUG:root:i=286 residual=0.03226678764413403
   DEBUG:root:i=287 residual=0.03195348106809185
   DEBUG:root:i=288 residual=0.031643257148155
   DEBUG:root:i=289 residual=0.03133608428345633
   DEBUG:root:i=290 residual=0.031031931237397704
   DEBUG:root:i=291 residual=0.03073076713217263
   DEBUG:root:i=292 residual=0.030432561443457757
   DEBUG:root:i=293 residual=0.030137283995123013
   DEBUG:root:i=294 residual=0.029844904954304443
   DEBUG:root:i=295 residual=0.02955539482616383
   DEBUG:root:i=296 residual=0.029268724449143246
   DEBUG:root:i=297 residual=0.028984864990101077
   DEBUG:root:i=298 residual=0.028703787939703188
   DEBUG:root:i=299 residual=0.028425465107695818
   DEBUG:root:i=300 residual=0.028149868618468715
   DEBUG:root:i=301 residual=0.02787697090662957
   DEBUG:root:i=302 residual=0.02760674471271365
   DEBUG:root:i=303 residual=0.027339163078802596
   DEBUG:root:i=304 residual=0.027074199344546156
   DEBUG:root:i=305 residual=0.026811827142950155
   DEBUG:root:i=306 residual=0.026552020396393757
   DEBUG:root:i=307 residual=0.02629475331281774
   DEBUG:root:i=308 residual=0.026040000381695107
   DEBUG:root:i=309 residual=0.025787736370410954
   DEBUG:root:i=310 residual=0.02553793632050474
   DEBUG:root:i=311 residual=0.02529057554400393
   DEBUG:root:i=312 residual=0.02504562961987279
   DEBUG:root:i=313 residual=0.024803074390562684
   DEBUG:root:i=314 residual=0.024562885958518358
   DEBUG:root:i=315 residual=0.024325040682839708
   DEBUG:root:i=316 residual=0.02408951517588877
   DEBUG:root:i=317 residual=0.023856286300180232
   DEBUG:root:i=318 residual=0.023625331165034374
   DEBUG:root:i=319 residual=0.02339662712355429
   DEBUG:root:i=320 residual=0.023170151769464215
   DEBUG:root:i=321 residual=0.022945882934107483
   DEBUG:root:i=322 residual=0.022723798683473524
   DEBUG:root:i=323 residual=0.022503877315258925
   DEBUG:root:i=324 residual=0.02228609735600343
   DEBUG:root:i=325 residual=0.022070437558222824
   DEBUG:root:i=326 residual=0.021856876897699044
   DEBUG:root:i=327 residual=0.021645394570674412
   DEBUG:root:i=328 residual=0.021435969991193408
   DEBUG:root:i=329 residual=0.02122858278844121
   DEBUG:root:i=330 residual=0.02102321280420985
   DEBUG:root:i=331 residual=0.02081984009019109
   DEBUG:root:i=332 residual=0.02061844490563043
   DEBUG:root:i=333 residual=0.020419007714729244
   DEBUG:root:i=334 residual=0.02022150918424416
   DEBUG:root:i=335 residual=0.020025930181080804
   DEBUG:root:i=336 residual=0.019832251769974073
   DEBUG:root:i=337 residual=0.019640455211059882
   DEBUG:root:i=338 residual=0.019450521957694426
   DEBUG:root:i=339 residual=0.019262433654135738
   DEBUG:root:i=340 residual=0.019076172133335415
   DEBUG:root:i=341 residual=0.018891719414779448
   DEBUG:root:i=342 residual=0.018709057702281782
   DEBUG:root:i=343 residual=0.018528169381880354
   DEBUG:root:i=344 residual=0.018349037019794463
   DEBUG:root:i=345 residual=0.018171643360328584
   DEBUG:root:i=346 residual=0.017995971323800874
   DEBUG:root:i=347 residual=0.01782200400462549
   DEBUG:root:i=348 residual=0.017649724669263653
   DEBUG:root:i=349 residual=0.017479116754354502
   DEBUG:root:i=350 residual=0.017310163864757897
   DEBUG:root:i=351 residual=0.017142849771658003
   DEBUG:root:i=352 residual=0.01697715841072271
   DEBUG:root:i=353 residual=0.016813073880265823
   DEBUG:root:i=354 residual=0.016650580439475277
   DEBUG:root:i=355 residual=0.01648966250652235
   DEBUG:root:i=356 residual=0.01633030465691622
   DEBUG:root:i=357 residual=0.016172491621739865
   DEBUG:root:i=358 residual=0.016016208285874873
   DEBUG:root:i=359 residual=0.015861439686396247
   DEBUG:root:i=360 residual=0.01570817101088563
   DEBUG:root:i=361 residual=0.015556387595740257
   DEBUG:root:i=362 residual=0.015406074924625167
   DEBUG:root:i=363 residual=0.01525721862679313
   DEBUG:root:i=364 residual=0.015109804475620835
   DEBUG:root:i=365 residual=0.014963818386906415
   DEBUG:root:i=366 residual=0.01481924641743718
   DEBUG:root:i=367 residual=0.014676074763459144
   DEBUG:root:i=368 residual=0.014534289759128198
   DEBUG:root:i=369 residual=0.014393877875104396
   DEBUG:root:i=370 residual=0.01425482571703331
   DEBUG:root:i=371 residual=0.014117120024118045
   DEBUG:root:i=372 residual=0.013980747667742443
   DEBUG:root:i=373 residual=0.013845695650016228
   DEBUG:root:i=374 residual=0.013711951102336521
   DEBUG:root:i=375 residual=0.01357950128419603
   DEBUG:root:i=376 residual=0.01344833358164559
   DEBUG:root:i=377 residual=0.013318435506049702
   DEBUG:root:i=378 residual=0.013189794692758721
   DEBUG:root:i=379 residual=0.013062398899784592
   DEBUG:root:i=380 residual=0.012936236006556577
   DEBUG:root:i=381 residual=0.012811294012549176
   DEBUG:root:i=382 residual=0.012687561036181776
   DEBUG:root:i=383 residual=0.012565025313441757
   DEBUG:root:i=384 residual=0.012443675196707997
   DEBUG:root:i=385 residual=0.012323499153599612
   DEBUG:root:i=386 residual=0.012204485765639692
   DEBUG:root:i=387 residual=0.012086623727202721
   DEBUG:root:i=388 residual=0.011969901844325812
   DEBUG:root:i=389 residual=0.011854309033458502
   DEBUG:root:i=390 residual=0.011739834320429694
   DEBUG:root:i=391 residual=0.011626466839291267
   DEBUG:root:i=392 residual=0.011514195831151219
   DEBUG:root:i=393 residual=0.011403010643173705
   DEBUG:root:i=394 residual=0.011292900727378615
   DEBUG:root:i=395 residual=0.011183855639643399
   DEBUG:root:i=396 residual=0.011075865038577312
   DEBUG:root:i=397 residual=0.010968918684561903
   DEBUG:root:i=398 residual=0.010863006438642658
   DEBUG:root:i=399 residual=0.010758118261462604
   DEBUG:root:i=400 residual=0.010654244212340073
   DEBUG:root:i=401 residual=0.010551374448218583
   DEBUG:root:i=402 residual=0.010449499222676652
   DEBUG:root:i=403 residual=0.010348608884928506
   DEBUG:root:i=404 residual=0.010248693878878126
   DEBUG:root:i=405 residual=0.010149744742179516
   DEBUG:root:i=406 residual=0.010051752105199216
   DEBUG:root:i=407 residual=0.009954706690194891
   DEBUG:root:i=408 residual=0.009858599310288357
   DEBUG:root:i=409 residual=0.009763420868610925
   DEBUG:root:i=410 residual=0.00966916235735081
   DEBUG:root:i=411 residual=0.009575814856908638
   DEBUG:root:i=412 residual=0.00948336953494222
   DEBUG:root:i=413 residual=0.009391817645555436
   DEBUG:root:i=414 residual=0.009301150528369663
   DEBUG:root:i=415 residual=0.009211359607659828
   DEBUG:root:i=416 residual=0.009122436391634804
   DEBUG:root:i=417 residual=0.009034372471401516
   DEBUG:root:i=418 residual=0.00894715952028818
   DEBUG:root:i=419 residual=0.008860789292907205
   DEBUG:root:i=420 residual=0.008775253624455675
   DEBUG:root:i=421 residual=0.008690544429797163
   DEBUG:root:i=422 residual=0.008606653702764823
   DEBUG:root:i=423 residual=0.0085235735152727
   DEBUG:root:i=424 residual=0.00844129601665727
   DEBUG:root:i=425 residual=0.008359813432772251
   DEBUG:root:i=426 residual=0.00827911806534496
   DEBUG:root:i=427 residual=0.008199202291124317
   DEBUG:root:i=428 residual=0.008120058561200497
   DEBUG:root:i=429 residual=0.008041679400270355
   DEBUG:root:i=430 residual=0.007964057405816355
   DEBUG:root:i=431 residual=0.007887185247484793
   DEBUG:root:i=432 residual=0.007811055666343505
   DEBUG:root:i=433 residual=0.007735661474120014
   DEBUG:root:i=434 residual=0.007660995552586578
   DEBUG:root:i=435 residual=0.0075870508527452465
   DEBUG:root:i=436 residual=0.0075138203943010485
   DEBUG:root:i=437 residual=0.007441297264848606
   DEBUG:root:i=438 residual=0.007369474619282511
   DEBUG:root:i=439 residual=0.007298345679042689
   DEBUG:root:i=440 residual=0.007227903731589153
   DEBUG:root:i=441 residual=0.007158142129599987
   DEBUG:root:i=442 residual=0.007089054290421251
   DEBUG:root:i=443 residual=0.007020633695446716
   DEBUG:root:i=444 residual=0.006952873889408113
   DEBUG:root:i=445 residual=0.006885768479802558
   DEBUG:root:i=446 residual=0.006819311136229915
   DEBUG:root:i=447 residual=0.006753495589847918
   DEBUG:root:i=448 residual=0.00668831563272057
   DEBUG:root:i=449 residual=0.0066237651172169655
   DEBUG:root:i=450 residual=0.0065598379554404045
   DEBUG:root:i=451 residual=0.006496528118596394
   DEBUG:root:i=452 residual=0.006433829636494087
   DEBUG:root:i=453 residual=0.006371736596866271
   DEBUG:root:i=454 residual=0.006310243144906125
   DEBUG:root:i=455 residual=0.00624934348258245
   DEBUG:root:i=456 residual=0.006189031868206697
   DEBUG:root:i=457 residual=0.006129302615797705
   DEBUG:root:i=458 residual=0.0060701500945705
   DEBUG:root:i=459 residual=0.006011568728355606
   DEBUG:root:i=460 residual=0.005953552995118497
   DEBUG:root:i=461 residual=0.005896097426417751
   DEBUG:root:i=462 residual=0.005839196606798174
   DEBUG:root:i=463 residual=0.005782845173406034
   DEBUG:root:i=464 residual=0.005727037815368225
   DEBUG:root:i=465 residual=0.005671769273358548
   DEBUG:root:i=466 residual=0.005617034338963586
   DEBUG:root:i=467 residual=0.005562827854376106
   DEBUG:root:i=468 residual=0.005509144711751591
   DEBUG:root:i=469 residual=0.005455979852799691
   DEBUG:root:i=470 residual=0.005403328268200456
   DEBUG:root:i=471 residual=0.005351184997259022
   DEBUG:root:i=472 residual=0.00529954512733228
   DEBUG:root:i=473 residual=0.005248403793391615
   DEBUG:root:i=474 residual=0.005197756177572057
   DEBUG:root:i=475 residual=0.005147597508652428
   DEBUG:root:i=476 residual=0.0050979230617172605
   DEBUG:root:i=477 residual=0.005048728157564351
   DEBUG:root:i=478 residual=0.005000008162352183
   DEBUG:root:i=479 residual=0.004951758487116158
   DEBUG:root:i=480 residual=0.004903974587391229
   DEBUG:root:i=481 residual=0.00485665196266016
   DEBUG:root:i=482 residual=0.004809786156046667
   DEBUG:root:i=483 residual=0.004763372753830224
   DEBUG:root:i=484 residual=0.004717407385018885
   DEBUG:root:i=485 residual=0.00467188572095269
   DEBUG:root:i=486 residual=0.0046268034749000035
   DEBUG:root:i=487 residual=0.004582156401619082
   DEBUG:root:i=488 residual=0.004537940297002529
   DEBUG:root:i=489 residual=0.004494150997595766
   DEBUG:root:i=490 residual=0.004450784380295911
   DEBUG:root:i=491 residual=0.004407836361884616
   DEBUG:root:i=492 residual=0.0043653028986400675
   DEBUG:root:i=493 residual=0.004323179986089972
   DEBUG:root:i=494 residual=0.004281463658389358
   DEBUG:root:i=495 residual=0.0042401499881522045
   DEBUG:root:i=496 residual=0.0041992350859854
   DEBUG:root:i=497 residual=0.004158715100141599
   DEBUG:root:i=498 residual=0.004118586216120392
   DEBUG:root:i=499 residual=0.004078844656326951
   DEBUG:root:i=500 residual=0.004039486679781429
   DEBUG:root:i=501 residual=0.004000508581620823
   DEBUG:root:i=502 residual=0.003961906692878642
   DEBUG:root:i=503 residual=0.003923677380078081
   DEBUG:root:i=504 residual=0.0038858170448475434
   DEBUG:root:i=505 residual=0.0038483221236897706
   DEBUG:root:i=506 residual=0.0038111890875378437
   DEBUG:root:i=507 residual=0.0037744144414624484
   DEBUG:root:i=508 residual=0.003737994724365153
   DEBUG:root:i=509 residual=0.0037019265085779223
   DEBUG:root:i=510 residual=0.003666206399654501
   DEBUG:root:i=511 residual=0.003630831035882923
   DEBUG:root:i=512 residual=0.00359579708814468
   DEBUG:root:i=513 residual=0.0035611012594724684
   DEBUG:root:i=514 residual=0.0035267402848261397
   DEBUG:root:i=515 residual=0.003492710930655757
   DEBUG:root:i=516 residual=0.003459009994766309
   DEBUG:root:i=517 residual=0.0034256343058850767
   DEBUG:root:i=518 residual=0.003392580723376141
   DEBUG:root:i=519 residual=0.0033598461370222083
   DEBUG:root:i=520 residual=0.0033274274666472603
   DEBUG:root:i=521 residual=0.0032953216618548183
   DEBUG:root:i=522 residual=0.003263525701762136
   DEBUG:root:i=523 residual=0.003232036594642869
   DEBUG:root:i=524 residual=0.0032008513777288935
   DEBUG:root:i=525 residual=0.003169967116886712
   DEBUG:root:i=526 residual=0.0031393809063381814
   DEBUG:root:i=527 residual=0.0031090898683904045
   DEBUG:root:i=528 residual=0.003079091153189989
   DEBUG:root:i=529 residual=0.0030493819384011587
   DEBUG:root:i=530 residual=0.003019959428959095
   DEBUG:root:i=531 residual=0.002990820856875243
   DEBUG:root:i=532 residual=0.00296196348081653
   DEBUG:root:i=533 residual=0.0029333845860115815
   DEBUG:root:i=534 residual=0.0029050814839434226
   DEBUG:root:i=535 residual=0.0028770515120133512
   DEBUG:root:i=536 residual=0.0028492920333872992
   DEBUG:root:i=537 residual=0.002821800436720349
   DEBUG:root:i=538 residual=0.002794574135872122
   DEBUG:root:i=539 residual=0.0027676105697365936
   DEBUG:root:i=540 residual=0.0027409072019015674
   DEBUG:root:i=541 residual=0.0027144615205022845
   DEBUG:root:i=542 residual=0.0026882710379687123
   DEBUG:root:i=543 residual=0.002662333290670243
   DEBUG:root:i=544 residual=0.00263664583887672
   DEBUG:root:i=545 residual=0.002611206266390627
   DEBUG:root:i=546 residual=0.0025860121803750534
   DEBUG:root:i=547 residual=0.0025610612110938654
   DEBUG:root:i=548 residual=0.002536351011706962
   DEBUG:root:i=549 residual=0.0025118792580328918
   DEBUG:root:i=550 residual=0.002487643648399714
   DEBUG:root:i=551 residual=0.0024636419033003463
   DEBUG:root:i=552 residual=0.00243987176530137
   DEBUG:root:i=553 residual=0.0024163309987483444
   DEBUG:root:i=554 residual=0.0023930173896254147
   DEBUG:root:i=555 residual=0.002369928745222449
   DEBUG:root:i=556 residual=0.002347062894077382
   DEBUG:root:i=557 residual=0.0023244176857015113
   DEBUG:root:i=558 residual=0.0023019909903288804
   DEBUG:root:i=559 residual=0.0022797806988445724
   DEBUG:root:i=560 residual=0.0022577847224183574
   DEBUG:root:i=561 residual=0.002236000992481643
   DEBUG:root:i=562 residual=0.002214427460371414
   DEBUG:root:i=563 residual=0.0021930620972444273
   DEBUG:root:i=564 residual=0.0021719028938509063
   DEBUG:root:i=565 residual=0.0021509478603637996
   DEBUG:root:i=566 residual=0.0021301950261276184
   DEBUG:root:i=567 residual=0.002109642439583821
   DEBUG:root:i=568 residual=0.0020892881679583234
   DEBUG:root:i=569 residual=0.0020691302971866713
   DEBUG:root:i=570 residual=0.002049166931698847
   DEBUG:root:i=571 residual=0.002029396194159741
   DEBUG:root:i=572 residual=0.0020098162254434666
   DEBUG:root:i=573 residual=0.0019904251843268135
   DEBUG:root:i=574 residual=0.0019712212474190098
   DEBUG:root:i=575 residual=0.0019522026088743636
   DEBUG:root:i=576 residual=0.001933367480347568
   DEBUG:root:i=577 residual=0.0019147140906913582
   DEBUG:root:i=578 residual=0.0018962406859267394
   DEBUG:root:i=579 residual=0.0018779455290047183
   DEBUG:root:i=580 residual=0.0018598268996147997
   DEBUG:root:i=581 residual=0.001841883094074219
   DEBUG:root:i=582 residual=0.0018241124251564247
   DEBUG:root:i=583 residual=0.001806513221931578
   DEBUG:root:i=584 residual=0.0017890838295949387
   DEBUG:root:i=585 residual=0.0017718226093025881
   DEBUG:root:i=586 residual=0.0017547279380785874
   DEBUG:root:i=587 residual=0.0017377982085774733
   DEBUG:root:i=588 residual=0.0017210318289865522
   DEBUG:root:i=589 residual=0.0017044272228906097
   DEBUG:root:i=590 residual=0.0016879828290293699
   DEBUG:root:i=591 residual=0.0016716971012878759
   DEBUG:root:i=592 residual=0.0016555685084252575
   DEBUG:root:i=593 residual=0.00163959553403141
   DEBUG:root:i=594 residual=0.001623776676294579
   DEBUG:root:i=595 residual=0.0016081104479098688
   DEBUG:root:i=596 residual=0.0015925953759586724
   DEBUG:root:i=597 residual=0.0015772300017297062
   DEBUG:root:i=598 residual=0.0015620128805889635
   DEBUG:root:i=599 residual=0.0015469425818304728
   DEBUG:root:i=600 residual=0.0015320176885944476
   DEBUG:root:i=601 residual=0.001517236797713557
   DEBUG:root:i=602 residual=0.00150259851949532
   DEBUG:root:i=603 residual=0.0014881014777383265
   DEBUG:root:i=604 residual=0.0014737443094761256
   DEBUG:root:i=605 residual=0.0014595256649305663
   DEBUG:root:i=606 residual=0.0014454442073577878
   DEBUG:root:i=607 residual=0.0014314986128844695
   DEBUG:root:i=608 residual=0.0014176875704472102
   DEBUG:root:i=609 residual=0.0014040097816093477
   DEBUG:root:i=610 residual=0.001390463960529231
   DEBUG:root:i=611 residual=0.0013770488336750535
   DEBUG:root:i=612 residual=0.001363763139944767
   DEBUG:root:i=613 residual=0.001350605630282704
   DEBUG:root:i=614 residual=0.0013375750677469542
   DEBUG:root:i=615 residual=0.0013246702273550447
   DEBUG:root:i=616 residual=0.0013118898959359152
   DEBUG:root:i=617 residual=0.0012992328719822824
   DEBUG:root:i=618 residual=0.0012866979656331041
   DEBUG:root:i=619 residual=0.0012742839985115534
   DEBUG:root:i=620 residual=0.0012619898036108246
   DEBUG:root:i=621 residual=0.0012498142251661814
   DEBUG:root:i=622 residual=0.001237756118593858
   DEBUG:root:i=623 residual=0.0012258143503732918
   DEBUG:root:i=624 residual=0.0012139877978648319
   DEBUG:root:i=625 residual=0.0012022753493133853
   DEBUG:root:i=626 residual=0.001190675903690709
   DEBUG:root:i=627 residual=0.001179188370596887
   DEBUG:root:i=628 residual=0.0011678116701367297
   DEBUG:root:i=629 residual=0.0011565447328554205
   DEBUG:root:i=630 residual=0.0011453864996161507
   DEBUG:root:i=631 residual=0.0011343359214951937
   DEBUG:root:i=632 residual=0.0011233919597236602
   DEBUG:root:i=633 residual=0.0011125535855149211
   DEBUG:root:i=634 residual=0.0011018197800546899
   DEBUG:root:i=635 residual=0.0010911895343048812
   DEBUG:root:i=636 residual=0.0010806618490510564
   DEBUG:root:i=637 residual=0.0010702357346390565
   DEBUG:root:i=638 residual=0.0010599102110275722
   DEBUG:root:i=639 residual=0.0010496843075831722
   DEBUG:root:i=640 residual=0.0010395570630835173
   DEBUG:root:i=641 residual=0.001029527525591296
   DEBUG:root:i=642 residual=0.001019594752258188
   DEBUG:root:i=643 residual=0.0010097578094798553
   DEBUG:root:i=644 residual=0.001000015772569806
   DEBUG:root:i=645 residual=0.0009903677257686043
   DEBUG:root:i=646 residual=0.000980812762213885
   DEBUG:root:i=647 residual=0.0009713499837014152
   DEBUG:root:i=648 residual=0.0009619785008071642
   DEBUG:root:i=649 residual=0.0009526974325743282
   DEBUG:root:i=650 residual=0.0009435059066703668
   DEBUG:root:i=651 residual=0.0009344030590738783
   DEBUG:root:i=652 residual=0.0009253880341772319
   DEBUG:root:i=653 residual=0.0009164599845387585
   DEBUG:root:i=654 residual=0.0009076180710471735
   DEBUG:root:i=655 residual=0.000898861462528429
   DEBUG:root:i=656 residual=0.0008901893359442622
   DEBUG:root:i=657 residual=0.0008816008761107151
   DEBUG:root:i=658 residual=0.00087309527580043
   DEBUG:root:i=659 residual=0.0008646717355087439
   DEBUG:root:i=660 residual=0.000856329463459458
   DEBUG:root:i=661 residual=0.0008480676755411858
   DEBUG:root:i=662 residual=0.000839885595176841
   DEBUG:root:i=663 residual=0.0008317824533280577
   DEBUG:root:i=664 residual=0.0008237574883560088
   DEBUG:root:i=665 residual=0.0008158099459465252
   DEBUG:root:i=666 residual=0.0008079390790789047
   DEBUG:root:i=667 residual=0.0008001441479566449
   DEBUG:root:i=668 residual=0.0007924244199359114
   DEBUG:root:i=669 residual=0.0007847791693891653
   DEBUG:root:i=670 residual=0.0007772076777553137
   DEBUG:root:i=671 residual=0.0007697092333548164
   DEBUG:root:i=672 residual=0.0007622831313955589
   DEBUG:root:i=673 residual=0.0007549286738910122
   DEBUG:root:i=674 residual=0.0007476451696069167
   DEBUG:root:i=675 residual=0.0007404319339168277
   DEBUG:root:i=676 residual=0.0007332882889052587
   DEBUG:root:i=677 residual=0.0007262135630953031
   DEBUG:root:i=678 residual=0.000719207091528008
   DEBUG:root:i=679 residual=0.0007122682156756519
   DEBUG:root:i=680 residual=0.0007053962833702373
   DEBUG:root:i=681 residual=0.0006985906486918866
   DEBUG:root:i=682 residual=0.000691850672004326
   DEBUG:root:i=683 residual=0.0006851757198053762
   DEBUG:root:i=684 residual=0.0006785651647474532
   DEBUG:root:i=685 residual=0.000672018385497485
   DEBUG:root:i=686 residual=0.0006655347667141719
   DEBUG:root:i=687 residual=0.0006591136990491319
   DEBUG:root:i=688 residual=0.0006527545789679045
   DEBUG:root:i=689 residual=0.00064645680880536
   DEBUG:root:i=690 residual=0.0006402197966304213
   DEBUG:root:i=691 residual=0.0006340429562621162
   DEBUG:root:i=692 residual=0.0006279257071497354
   DEBUG:root:i=693 residual=0.0006218674743397775
   DEBUG:root:i=694 residual=0.0006158676884438254
   DEBUG:root:i=695 residual=0.0006099257855518362
   DEBUG:root:i=696 residual=0.0006040412072203594
   DEBUG:root:i=697 residual=0.0005982134003562459
   DEBUG:root:i=698 residual=0.0005924418172281542
   DEBUG:root:i=699 residual=0.0005867259154006227
   DEBUG:root:i=700 residual=0.0005810651576527512
   DEBUG:root:i=701 residual=0.0005754590119507842
   DEBUG:root:i=702 residual=0.0005699069513915686
   DEBUG:root:i=703 residual=0.0005644084541745049
   DEBUG:root:i=704 residual=0.000558963003517225
   DEBUG:root:i=705 residual=0.0005535700876123071
   DEBUG:root:i=706 residual=0.0005482291996148847
   DEBUG:root:i=707 residual=0.0005429398375621742
   DEBUG:root:i=708 residual=0.0005377015043558942
   DEBUG:root:i=709 residual=0.0005325137076324494
   DEBUG:root:i=710 residual=0.0005273759598434239
   DEBUG:root:i=711 residual=0.0005222877781158076
   DEBUG:root:i=712 residual=0.0005172486842449395
   DEBUG:root:i=713 residual=0.000512258204618152
   DEBUG:root:i=714 residual=0.0005073158702416201
   DEBUG:root:i=715 residual=0.0005024212165966024
   DEBUG:root:i=716 residual=0.0004975737836703165
   DEBUG:root:i=717 residual=0.0004927731158710195
   DEBUG:root:i=718 residual=0.0004880187620297902
   DEBUG:root:i=719 residual=0.00048331027531085097
   DEBUG:root:i=720 residual=0.00047864721320844944
   DEBUG:root:i=721 residual=0.0004740291374521637
   DEBUG:root:i=722 residual=0.0004694556140429326
   DEBUG:root:i=723 residual=0.0004649262131342885
   DEBUG:root:i=724 residual=0.00046044050903417614
   DEBUG:root:i=725 residual=0.00045599808020276863
   DEBUG:root:i=726 residual=0.00045159850908426367
   DEBUG:root:i=727 residual=0.0004472413822173419
   DEBUG:root:i=728 residual=0.0004429262901241639
   DEBUG:root:i=729 residual=0.00043865282722620907
   DEBUG:root:i=730 residual=0.00043442059192984523
   DEBUG:root:i=731 residual=0.0004302291864771149
   DEBUG:root:i=732 residual=0.0004260782169390025
   DEBUG:root:i=733 residual=0.000421967293203467
   DEBUG:root:i=734 residual=0.0004178960289342042
   DEBUG:root:i=735 residual=0.00041386404150001334
   DEBUG:root:i=736 residual=0.00040987095196334995
   DEBUG:root:i=737 residual=0.000405916385062907
   DEBUG:root:i=738 residual=0.00040199996911443063
   DEBUG:root:i=739 residual=0.00039812133609654993
   DEBUG:root:i=740 residual=0.0003942801214531019
   DEBUG:root:i=741 residual=0.00039047596422303317
   DEBUG:root:i=742 residual=0.0003867085068338405
   DEBUG:root:i=743 residual=0.00038297739525490026
   DEBUG:root:i=744 residual=0.00037928227885962164
   DEBUG:root:i=745 residual=0.000375622810316008
   DEBUG:root:i=746 residual=0.0003719986457598699
   DEBUG:root:i=747 residual=0.0003684094445593082
   DEBUG:root:i=748 residual=0.00036485486942949944
   DEBUG:root:i=749 residual=0.00036133458629463626
   DEBUG:root:i=750 residual=0.0003578482643147106
   DEBUG:root:i=751 residual=0.00035439557585516235
   DEBUG:root:i=752 residual=0.0003509761964498095
   DEBUG:root:i=753 residual=0.00034758980471279296
   DEBUG:root:i=754 residual=0.0003442360824131381
   DEBUG:root:i=755 residual=0.0003409147143619886
   DEBUG:root:i=756 residual=0.0003376253884344923
   DEBUG:root:i=757 residual=0.00033436779550030047
   DEBUG:root:i=758 residual=0.0003311416293925943
   DEBUG:root:i=759 residual=0.00032794658693470024
   DEBUG:root:i=760 residual=0.00032478236787591037
   DEBUG:root:i=761 residual=0.00032164867482180017
   DEBUG:root:i=762 residual=0.0003185452132849499
   DEBUG:root:i=763 residual=0.0003154716915915312
   DEBUG:root:i=764 residual=0.0003124278209138276
   DEBUG:root:i=765 residual=0.00030941331517553
   DEBUG:root:i=766 residual=0.0003064278911040059
   DEBUG:root:i=767 residual=0.000303471268106811
   DEBUG:root:i=768 residual=0.00030054316834793126
   DEBUG:root:i=769 residual=0.0002976433166344251
   DEBUG:root:i=770 residual=0.0002947714404536144
   DEBUG:root:i=771 residual=0.0002919272699322563
   DEBUG:root:i=772 residual=0.00028911053774308213
   DEBUG:root:i=773 residual=0.0002863209791762168
   DEBUG:root:i=774 residual=0.0002835583320929934
   DEBUG:root:i=775 residual=0.00028082233686226504
   DEBUG:root:i=776 residual=0.0002781127363874394
   DEBUG:root:i=777 residual=0.00027542927596870985
   DEBUG:root:i=778 residual=0.0002727717034726691
   DEBUG:root:i=779 residual=0.00027013976912507556
   DEBUG:root:i=780 residual=0.0002675332255932773
   DEBUG:root:i=781 residual=0.00026495182791660857
   DEBUG:root:i=782 residual=0.0002623953334844251
   DEBUG:root:i=783 residual=0.0002598635020604081
   DEBUG:root:i=784 residual=0.0002573560957317315
   DEBUG:root:i=785 residual=0.0002548728788296976
   DEBUG:root:i=786 residual=0.0002524136180067488
   DEBUG:root:i=787 residual=0.0002499780821372994
   DEBUG:root:i=788 residual=0.000247566042331426
   DEBUG:root:i=789 residual=0.0002451772719234607
   DEBUG:root:i=790 residual=0.0002428115464414774
   DEBUG:root:i=791 residual=0.00024046864353842446
   DEBUG:root:i=792 residual=0.0002381483430579062
   DEBUG:root:i=793 residual=0.00023585042692187623
   DEBUG:root:i=794 residual=0.0002335746791876151
   DEBUG:root:i=795 residual=0.00023132088598267633
   DEBUG:root:i=796 residual=0.00022908883552892696
   DEBUG:root:i=797 residual=0.0002268783180525035
   DEBUG:root:i=798 residual=0.00022468912578867416
   DEBUG:root:i=799 residual=0.0002225210530525637
   DEBUG:root:i=800 residual=0.00022037389608343076
   DEBUG:root:i=801 residual=0.00021824745308564534
   DEBUG:root:i=802 residual=0.0002161415242292361
   DEBUG:root:i=803 residual=0.00021405591160370948
   DEBUG:root:i=804 residual=0.0002119904192127998
   DEBUG:root:i=805 residual=0.0002099448529775598
   DEBUG:root:i=806 residual=0.00020791902062870313
   DEBUG:root:i=807 residual=0.00020591273180231025
   DEBUG:root:i=808 residual=0.00020392579794667294
   DEBUG:root:i=809 residual=0.00020195803236520084
   DEBUG:root:i=810 residual=0.00020000925010243997
   DEBUG:root:i=811 residual=0.00019807926802676036
   DEBUG:root:i=812 residual=0.00019616790478012468
   DEBUG:root:i=813 residual=0.00019427498074803522
   DEBUG:root:i=814 residual=0.00019240031801912968
   DEBUG:root:i=815 residual=0.0001905437404140054
   DEBUG:root:i=816 residual=0.00018870507350314685
   DEBUG:root:i=817 residual=0.0001868841444648802
   DEBUG:root:i=818 residual=0.00018508078219181022
   DEBUG:root:i=819 residual=0.00018329481719681466
   DEBUG:root:i=820 residual=0.00018152608164634287
   DEBUG:root:i=821 residual=0.0001797744093497727
   DEBUG:root:i=822 residual=0.00017803963564657109
   DEBUG:root:i=823 residual=0.0001763215975512848
   DEBUG:root:i=824 residual=0.0001746201335916108
   DEBUG:root:i=825 residual=0.00017293508388118171
   DEBUG:root:i=826 residual=0.00017126629006448819
   DEBUG:root:i=827 residual=0.00016961359532103841
   DEBUG:root:i=828 residual=0.00016797684431605036
   DEBUG:root:i=829 residual=0.00016635588327195626
   DEBUG:root:i=830 residual=0.00016475055983667663
   DEBUG:root:i=831 residual=0.00016316072315523895
   DEBUG:root:i=832 residual=0.0001615862238257125
   DEBUG:root:i=833 residual=0.00016002691389778006
   DEBUG:root:i=834 residual=0.0001584826467991459
   DEBUG:root:i=835 residual=0.00015695327744156412
   DEBUG:root:i=836 residual=0.00015543866207911396
   DEBUG:root:i=837 residual=0.00015393865840467375
   DEBUG:root:i=838 residual=0.00015245312543186253
   DEBUG:root:i=839 residual=0.00015098192357784429
   DEBUG:root:i=840 residual=0.00014952491456389423
   DEBUG:root:i=841 residual=0.00014808196149403055
   DEBUG:root:i=842 residual=0.00014665292874539875
   DEBUG:root:i=843 residual=0.00014523768204199012
   DEBUG:root:i=844 residual=0.0001438360883902791
   DEBUG:root:i=845 residual=0.0001424480160431096
   DEBUG:root:i=846 residual=0.00014107333458361518
   DEBUG:root:i=847 residual=0.00013971191483543792
   DEBUG:root:i=848 residual=0.0001383636288625018
   DEBUG:root:i=849 residual=0.00013702834993438722
   DEBUG:root:i=850 residual=0.00013570595259308405
   DEBUG:root:i=851 residual=0.00013439631254595616
   DEBUG:root:i=852 residual=0.00013309930673252834
   DEBUG:root:i=853 residual=0.0001318148132755544
   DEBUG:root:i=854 residual=0.00013054271144110762
   DEBUG:root:i=855 residual=0.0001292828817256451
   DEBUG:root:i=856 residual=0.00012803520569943362
   DEBUG:root:i=857 residual=0.00012679956614209087
   DEBUG:root:i=858 residual=0.00012557584691077074
   DEBUG:root:i=859 residual=0.00012436393303649103
   DEBUG:root:i=860 residual=0.00012316371057788898
   DEBUG:root:i=861 residual=0.00012197506681474541
   DEBUG:root:i=862 residual=0.00012079789000433606
   DEBUG:root:i=863 residual=0.00011963206954107165
   DEBUG:root:i=864 residual=0.00011847749585412622
   DEBUG:root:i=865 residual=0.00011733406045009428
   DEBUG:root:i=866 residual=0.00011620165586882329
   DEBUG:root:i=867 residual=0.00011508017569740918
   DEBUG:root:i=868 residual=0.00011396951455505601
   DEBUG:root:i=869 residual=0.0001128695680677981
   DEBUG:root:i=870 residual=0.00011178023286306788
   DEBUG:root:i=871 residual=0.00011070140657930443
   DEBUG:root:i=872 residual=0.0001096329878213278
   DEBUG:root:i=873 residual=0.00010857487619806551
   DEBUG:root:i=874 residual=0.00010752697227927281
   DEBUG:root:i=875 residual=0.0001064891775695164
   DEBUG:root:i=876 residual=0.00010546139456709464
   DEBUG:root:i=877 residual=0.00010444352667965151
   DEBUG:root:i=878 residual=0.00010343547825387142
   DEBUG:root:i=879 residual=0.00010243715454932616
   DEBUG:root:i=880 residual=0.0001014484617784071
   DEBUG:root:i=881 residual=0.00010046930698737381
   DEBUG:root:i=882 residual=9.949959820335562e-05
   DEBUG:root:i=883 residual=9.853924427686696e-05
   DEBUG:root:i=884 residual=9.758815497416757e-05
   DEBUG:root:i=885 residual=9.664624090400251e-05
   DEBUG:root:i=886 residual=9.57134135604563e-05
   DEBUG:root:i=887 residual=9.478958527911824e-05
   DEBUG:root:i=888 residual=9.38746692553772e-05
   DEBUG:root:i=889 residual=9.296857948631525e-05
   DEBUG:root:i=890 residual=9.20712308668322e-05
   DEBUG:root:i=891 residual=9.1182539023865e-05
   DEBUG:root:i=892 residual=9.030242047231235e-05
   DEBUG:root:i=893 residual=8.943079249165031e-05
   DEBUG:root:i=894 residual=8.856757318002854e-05
   DEBUG:root:i=895 residual=8.771268141909628e-05
   DEBUG:root:i=896 residual=8.686603686429439e-05
   DEBUG:root:i=897 residual=8.602755997825349e-05
   DEBUG:root:i=898 residual=8.519717191328208e-05
   DEBUG:root:i=899 residual=8.437479468613301e-05
   DEBUG:root:i=900 residual=8.356035100794749e-05
   DEBUG:root:i=901 residual=8.275376431186317e-05
   DEBUG:root:i=902 residual=8.195495884131499e-05
   DEBUG:root:i=903 residual=8.116385952203578e-05
   DEBUG:root:i=904 residual=8.038039197200681e-05
   DEBUG:root:i=905 residual=7.960448260804647e-05
   DEBUG:root:i=906 residual=7.883605851374409e-05
   DEBUG:root:i=907 residual=7.807504744856209e-05
   DEBUG:root:i=908 residual=7.732137794087429e-05
   DEBUG:root:i=909 residual=7.657497912547535e-05
   DEBUG:root:i=910 residual=7.583578089043927e-05
   DEBUG:root:i=911 residual=7.510371375473209e-05
   DEBUG:root:i=912 residual=7.437870893041411e-05
   DEBUG:root:i=913 residual=7.366069828915955e-05
   DEBUG:root:i=914 residual=7.294961434127813e-05
   DEBUG:root:i=915 residual=7.224539029975448e-05
   DEBUG:root:i=916 residual=7.154795994791978e-05
   DEBUG:root:i=917 residual=7.08572577749762e-05
   DEBUG:root:i=918 residual=7.017321884961272e-05
   DEBUG:root:i=919 residual=6.949577890233865e-05
   DEBUG:root:i=920 residual=6.882487428090232e-05
   DEBUG:root:i=921 residual=6.816044191629396e-05
   DEBUG:root:i=922 residual=6.750241939032843e-05
   DEBUG:root:i=923 residual=6.685074488625741e-05
   DEBUG:root:i=924 residual=6.620535711169878e-05
   DEBUG:root:i=925 residual=6.556619545718008e-05
   DEBUG:root:i=926 residual=6.49331998465158e-05
   DEBUG:root:i=927 residual=6.430631079348163e-05
   DEBUG:root:i=928 residual=6.36854693884583e-05
   DEBUG:root:i=929 residual=6.307061730122545e-05
   DEBUG:root:i=930 residual=6.246169671858243e-05
   DEBUG:root:i=931 residual=6.185865045550417e-05
   DEBUG:root:i=932 residual=6.126142185254836e-05
   DEBUG:root:i=933 residual=6.0669954736744766e-05
   DEBUG:root:i=934 residual=6.0084193562453164e-05
   DEBUG:root:i=935 residual=5.9504083279482734e-05
   DEBUG:root:i=936 residual=5.892956936187427e-05
   DEBUG:root:i=937 residual=5.836059784184526e-05
   DEBUG:root:i=938 residual=5.779711523176767e-05
   DEBUG:root:i=939 residual=5.723906859172176e-05
   DEBUG:root:i=940 residual=5.668640546104941e-05
   DEBUG:root:i=941 residual=5.613907393090267e-05
   DEBUG:root:i=942 residual=5.559702255234609e-05
   DEBUG:root:i=943 residual=5.5060200392855235e-05
   DEBUG:root:i=944 residual=5.452855700886533e-05
   DEBUG:root:i=945 residual=5.400204242403245e-05
   DEBUG:root:i=946 residual=5.348060717897557e-05
   DEBUG:root:i=947 residual=5.2964202281605e-05
   DEBUG:root:i=948 residual=5.245277917302894e-05
   DEBUG:root:i=949 residual=5.194628981513438e-05
   DEBUG:root:i=950 residual=5.144468661175954e-05
   DEBUG:root:i=951 residual=5.0947922409126076e-05
   DEBUG:root:i=952 residual=5.045595055337882e-05
   DEBUG:root:i=953 residual=4.99687247838213e-05
   DEBUG:root:i=954 residual=4.9486199329337636e-05
   DEBUG:root:i=955 residual=4.900832884085862e-05
   DEBUG:root:i=956 residual=4.8535068404579454e-05
   DEBUG:root:i=957 residual=4.806637358641261e-05
   DEBUG:root:i=958 residual=4.7602200299945794e-05
   DEBUG:root:i=959 residual=4.714250494274429e-05
   DEBUG:root:i=960 residual=4.668724431760107e-05
   DEBUG:root:i=961 residual=4.623637562325208e-05
   DEBUG:root:i=962 residual=4.578985652170344e-05
   DEBUG:root:i=963 residual=4.5347645016954756e-05
   DEBUG:root:i=964 residual=4.490969957576836e-05
   DEBUG:root:i=965 residual=4.447597905492349e-05
   DEBUG:root:i=966 residual=4.404644268076871e-05
   DEBUG:root:i=967 residual=4.362105007955223e-05
   DEBUG:root:i=968 residual=4.319976130986821e-05
   DEBUG:root:i=969 residual=4.2782536726554834e-05
   DEBUG:root:i=970 residual=4.236933716587367e-05
   DEBUG:root:i=971 residual=4.1960123803902085e-05
   DEBUG:root:i=972 residual=4.155485815594341e-05
   DEBUG:root:i=973 residual=4.11535021495893e-05
   DEBUG:root:i=974 residual=4.0756018085827094e-05
   DEBUG:root:i=975 residual=4.036236857506058e-05
   DEBUG:root:i=976 residual=3.9972516678313845e-05
   DEBUG:root:i=977 residual=3.9586425707418756e-05
   DEBUG:root:i=978 residual=3.92040594001578e-05
   DEBUG:root:i=979 residual=3.8825381846588435e-05
   DEBUG:root:i=980 residual=3.8450357435743465e-05
   DEBUG:root:i=981 residual=3.807895092372183e-05
   DEBUG:root:i=982 residual=3.77111274278671e-05
   DEBUG:root:i=983 residual=3.7346852378619847e-05
   DEBUG:root:i=984 residual=3.6986091514043706e-05
   DEBUG:root:i=985 residual=3.6628810977335805e-05
   DEBUG:root:i=986 residual=3.6274977179805096e-05
   DEBUG:root:i=987 residual=3.592455684500082e-05
   DEBUG:root:i=988 residual=3.5577517076554115e-05
   DEBUG:root:i=989 residual=3.523382524066396e-05
   DEBUG:root:i=990 residual=3.489344905937276e-05
   DEBUG:root:i=991 residual=3.455635653637501e-05
   DEBUG:root:i=992 residual=3.422251598165436e-05
   DEBUG:root:i=993 residual=3.389189604043546e-05
   DEBUG:root:i=994 residual=3.3564465644578776e-05
   DEBUG:root:i=995 residual=3.324019403618439e-05
   DEBUG:root:i=996 residual=3.2919050694829215e-05
   DEBUG:root:i=997 residual=3.260100549805292e-05
   DEBUG:root:i=998 residual=3.2286028525958683e-05
   DEBUG:root:i=999 residual=3.197409018809342e-05
   INFO:root:rank=0 pagerank=5.2386e+01 url=www.lawfareblog.com/0-days-n-days-iphones-and-android
   INFO:root:rank=1 pagerank=5.2386e+01 url=www.lawfareblog.com/lawfare-job-board
   INFO:root:rank=2 pagerank=7.9441e+00 url=www.lawfareblog.com/doj-charges-two-former-twitter-employees-spying-saudis
   INFO:root:rank=3 pagerank=2.3700e+00 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban
   INFO:root:rank=4 pagerank=1.5534e+00 url=www.lawfareblog.com/subscribe-lawfare
   INFO:root:rank=5 pagerank=1.1875e+00 url=www.lawfareblog.com/lessons-so-far-whatsapp-v-nso
   INFO:root:rank=6 pagerank=1.1875e+00 url=www.lawfareblog.com/cyberlaw-podcast-plumbing-depths-artificial-stupidity
   INFO:root:rank=7 pagerank=1.1875e+00 url=www.lawfareblog.com/our-comments-policy
   INFO:root:rank=8 pagerank=1.1875e+00 url=www.lawfareblog.com/upcoming-events
   INFO:root:rank=9 pagerank=1.1875e+00 url=www.lawfareblog.com/cyberlaw-podcast-sandworm-and-grus-global-intifada

   ```

   Task 2, part 1:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona'
   INFO:gensim.models.keyedvectors:loading projection weights from /home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz
   INFO:gensim.utils:KeyedVectors lifecycle event {'msg': 'loaded (3000000, 300) matrix of type float32 from /home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz', 'binary': True, 'encoding': 'utf8', 'datetime': '2022-12-11T20:54:49.669643', 'gensim': '4.2.0', 'python': '3.6.9 (default, Dec  8 2021, 21:08:43) \n[GCC 8.4.0]', 'platform': 'Linux-4.15.0-88-generic-x86_64-with-Ubuntu-18.04-bionic', 'event': 'load_word2vec_format'}
   INFO:root:rank=0 pagerank=8.8870e-01 url=www.lawfareblog.com/0-days-n-days-iphones-and-android
   INFO:root:rank=1 pagerank=8.8867e-01 url=www.lawfareblog.com/lawfare-job-board
   INFO:root:rank=2 pagerank=1.8256e-01 url=www.lawfareblog.com/doj-charges-two-former-twitter-employees-spying-saudis
   INFO:root:rank=3 pagerank=1.4907e-01 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban
   INFO:root:rank=4 pagerank=1.4907e-01 url=www.lawfareblog.com/subscribe-lawfare
   INFO:root:rank=5 pagerank=1.0729e-01 url=www.lawfareblog.com/lessons-so-far-whatsapp-v-nso
   INFO:root:rank=6 pagerank=1.0199e-01 url=www.lawfareblog.com/cyberlaw-podcast-plumbing-depths-artificial-stupidity
   INFO:root:rank=7 pagerank=1.0199e-01 url=www.lawfareblog.com/our-comments-policy
   INFO:root:rank=8 pagerank=9.4298e-02 url=www.lawfareblog.com/upcoming-events
   INFO:root:rank=9 pagerank=8.7207e-02 url=www.lawfareblog.com/cyberlaw-podcast-sandworm-and-grus-global-intifada

   ```

   Task 2, part 2:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona' --search_query='-corona'
   INFO:gensim.models.keyedvectors:loading projection weights from /home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz
   INFO:gensim.utils:KeyedVectors lifecycle event {'msg': 'loaded (3000000, 300) matrix of type float32 from /home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz', 'binary': True, 'encoding': 'utf8', 'datetime': '2022-12-11T20:54:49.669643', 'gensim': '4.2.0', 'python': '3.6.9 (default, Dec  8 2021, 21:08:43) \n[GCC 8.4.0]', 'platform': 'Linux-4.15.0-88-generic-x86_64-with-Ubuntu-18.04-bionic', 'event': 'load_word2vec_format'}
   INFO:root:rank=0 pagerank=8.8870e-01 url=www.lawfareblog.com/0-days-n-days-iphones-and-android
   INFO:root:rank=1 pagerank=8.8867e-01 url=www.lawfareblog.com/lawfare-job-board
   INFO:root:rank=2 pagerank=1.8256e-01 url=www.lawfareblog.com/doj-charges-two-former-twitter-employees-spying-saudis
   INFO:root:rank=3 pagerank=1.4907e-01 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban
   INFO:root:rank=4 pagerank=1.4907e-01 url=www.lawfareblog.com/subscribe-lawfare
   INFO:root:rank=5 pagerank=1.0729e-01 url=www.lawfareblog.com/lessons-so-far-whatsapp-v-nso
   INFO:root:rank=6 pagerank=1.0199e-01 url=www.lawfareblog.com/cyberlaw-podcast-plumbing-depths-artificial-stupidity
   INFO:root:rank=7 pagerank=1.0199e-01 url=www.lawfareblog.com/our-comments-policy
   INFO:root:rank=8 pagerank=9.4298e-02 url=www.lawfareblog.com/upcoming-events
   INFO:root:rank=9 pagerank=8.7207e-02 url=www.lawfareblog.com/cyberlaw-podcast-sandworm-and-grus-global-intifada

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





#Pagerank 2.0 outputs
```
chuckrak@Chucks-MacBook-Pro-2 pagerank % python3 pagerank.py --data=./data/lawfareblog.csv.gz --search_query='weapons'
INFO:gensim.models.keyedvectors:loading projection weights from /Users/chuckrak/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz
INFO:gensim.utils:KeyedVectors lifecycle event {'msg': 'loaded (3000000, 300) matrix of type float32 from /Users/chuckrak/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz', 'binary': True, 'encoding': 'utf8', 'datetime': '2022-12-05T23:49:29.473600', 'gensim': '4.2.0', 'python': '3.10.8 (main, Oct 13 2022, 10:17:43) [Clang 14.0.0 (clang-1400.0.29.102)]', 'platform': 'macOS-13.0.1-x86_64-i386-64bit', 'event': 'load_word2vec_format'}
INFO:root:rank=0 pagerank=6.6253e-02 url=www.lawfareblog.com/donald-trump-and-politically-weaponized-executive-branch
INFO:root:rank=1 pagerank=5.6646e-03 url=www.lawfareblog.com/slaughterbots-and-other-anticipated-autonomous-weapons-problems
INFO:root:rank=2 pagerank=3.2004e-03 url=www.lawfareblog.com/introducing-new-paper-weaponized-interdependence
INFO:root:rank=3 pagerank=3.1662e-03 url=www.lawfareblog.com/atomwaffen-division-member-pleads-guilty-firearms-charge
INFO:root:rank=4 pagerank=2.3146e-03 url=www.lawfareblog.com/history-do-it-yourself-weapons-and-explosives-manuals-america
INFO:root:rank=5 pagerank=2.2856e-03 url=www.lawfareblog.com/explainable-ai-and-legality-autonomous-weapon-systems
INFO:root:rank=6 pagerank=2.2713e-03 url=www.lawfareblog.com/right-wing-extremists-new-weapon
INFO:root:rank=7 pagerank=2.2654e-03 url=www.lawfareblog.com/lethal-autonomous-weapons-systems-first-and-second-un-gge-meetings
INFO:root:rank=8 pagerank=2.2236e-03 url=www.lawfareblog.com/lethal-autonomous-weapons-systems-recent-developments
INFO:root:rank=9 pagerank=2.1573e-03 url=www.lawfareblog.com/critical-gaps-remain-defense-department-weapons-system-cybersecurity
```
