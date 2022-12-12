



cerb2020@lambda-server:~/final$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='corona'
INFO:gensim.models.keyedvectors:loading projection weights from /home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz
INFO:gensim.utils:KeyedVectors lifecycle event {'msg': 'loaded (3000000, 300) matrix of type float32 from /home/cerb2020/gensim-data/word2vec-google-news-300/word2vec-google-news-300.gz', 'binary': True, 'encoding': 'utf8', 'datetime': '2022-12-07T13:09:08.334534', 'gensim': '4.2.0', 'python': '3.6.9 (default, Dec  8 2021, 21:08:43) \n[GCC 8.4.0]', 'platform': 'Linux-4.15.0-88-generic-x86_64-with-Ubuntu-18.04-bionic', 'event': 'load_word2vec_format'}
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

cerb2020@lambda-server:~/final$ python3 pagerank.py --data=./data/lawfareblog.csv.gz --search_query='trump'
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


cerb2020@lambda-server:~/final$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='iran'
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