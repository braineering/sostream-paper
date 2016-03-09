# Real-time Analysis of Social Networks Leveraging the Flink Framework
*ACM DEBS Grand Challenge 2016*
- - -

In this paper, we present a solution to the DEBS 2016 Grand Challenge that leverages Apache Flink, an open source platform for distributed stream and batch processing. The challenge focuses on the real-time analysis of an evolving social-network graph, to
(1) determine the posts that trigger the current most activity, and
(2) identify large communities that are currently involved in a topic.

We design the system architecture focusing on the exploitation of parallelism and memory efficiency so to enable an effective processing of high volume data streams on a distributed infrastructure.

Our solution to the first query relies on a distributed and fine-grain approach for updating the post scores and determining partial ranks, which are then merged into a single final rank. Furthermore, changes in the final rank are efficiently identified so to update the output only if needed.

For the second query, we plan to exploit the benefits of concurrent batch and stream processing as envisioned by the lambda architecture as well as the support provided by the Flink graph processing API.

## Grand Challlenge Problem
The ACM DEBS 2016 Grand Challenge is the sixth in a series of challenges which seek to provide a common ground and uniform evaluation criteria for a competition aimed at both research and industrial event-based systems. The goal of the 2016 DEBS Grand Challenge competition is to evaluate event-based systems for real-time analytics over high volume data streams in the context of graph models.

The underlying scenario addresses the analysis metrics for an evolving social-network graph. Specifically, the 2016 Grand Challenge targets the following problems: (1) identification of the posts that currently trigger the most activity, and (2) identification of large communities that are currently involved in a topic. The corresponding queries require continuous analysis of an evolving graph structure under the consideration of multiple streams that reflect updates to the graph.

The data for the DEBS 2016 Grand Challenge is based on the dataset provided together with the [LDBC](http://ldbcouncil.org/) Social Network Benchmark. It takes up the general scenario from the 2014 SIGMOD Programming contest but - in contrasts to the SIGMOD contest - explicitly focuses on processing streaming data. Details about the data, the queries for the Grand Challenge, and information about how the challenge will be evaluated are provided below.

### Input Data Streams
The input data is organized in four separate streams, each provided as a text file. Namely, we provide the following input data files:

* **fiendship.dat**: (ts, user_id_1, user_id_2), where *ts* is the friendship's establishment timestamp, *user_id_1* is the id of one of the users, *user_id_2* is the id of the other user.

* **posts.dat**: (ts, post_id, user_id, post, user), where *ts* is the post's timestamp, *post_id* is the unique id of the post, *user_id* is the unique id of the user, *post* is a string containing the actual post content, *user* is a string containing the actual user name.

* **comments.dat**: (ts, comment_id, user_id, comment, user, comment_replied, post_commented2), where *ts* is the comment's timestamp, *comment_id* is the unique id of the comment, *user_id* is the unique id of the user, *comment* is a string containing the actual comment, *user* is a string containing the actual user name, *comment_replied* is the id of the comment being replied to (-1 if the tuple is a reply to a post), *post_commented* is the id of the post being commented (-1 if the tuple is a reply to a comment).

* **likes.dat**: (ts, user_id, comment_id), where *ts* is the like's timestamp, *user_id* is the id of the user liking the comment, *comment_id* is the id of the comment.

Please note that as the contents of files represent data streams, each file is sorted based on its respective timestamp field.

A sample set of input data streams can be downloaded from [here](https://www.dropbox.com/s/vo83ohrgcgfqq27/data.tar.bz2). These files are only meant for development and debugging. Our tests will be based on other files (possibly of larger size), but strictly following the same format.

### Query 1
The goal of query 1 is to compute the top-3 scoring active posts, producing an updated result every time they change.

The total score of an active post P is computed as the sum of its own score plus the score of all its related comments. Active posts having the same total score should be ranked based on their timestamps (in descending order), and if their timestamps are also the same, they should be ranked based on the timestamps of their last received related comments (in descending order). A comment C is related to a post P if it is a direct reply to P or if the chain of C's preceding messages links back to P.

Each new post has an initial own score of 10 which decreases by 1 each time another 24 hours elapse since the post's creation. Each new comment's score is also initially set to 10 and decreases by 1 in the same way (every 24 hours since the comment's creation). Both post and comment scores are non-negative numbers, that is, they cannot drop below zero. A post is considered no longer active (that is, no longer part of the present and future analysis) as soon as its total score reaches zero, even if it receives additional comments in the future.

#### Output specification
(ts,top1_post_id,top1_post_user,top1_post_score,top1_post_commenters,
top2_post_id,top2_post_user,top2_post_score,top2_post_commenters,
top3_post_id,top3_post_user,top3_post_score,top3_post_commenters)

ts: the timestamp of the tuple that triggers a change in the top-3 scoring active posts appearing in the rest of the tuple
topX_post_id: the unique id of the top-X post
topX_post_user: the user author of top-X post
topX_post_commenters: the number of commenters (excluding the post author) for the top-X post

Results should be sorted by their timestamp field. The character "-" (a minus sign without the quotations) should be used for each of the fields (post id, post user, post commenters) of any of the top-3 positions that has not been defined. Needless to say, the logical time of the query advances based on the timestamps of the input tuples, not the system clock.
Sample output tuples for the query

2010-09-19 12:33:01,25769805561,Karl Fischer,115,10,25769805933,Chong Liu,83,4,25769804888,-,-,- 2010-10-09 21:55:24,34359739095,Karl Fischer,58,7,34359740594,Paul Becker,40,2,34359740220,Chong Zhang,10,0 2010-12-27 22:11:54,42949673675,Anson Chen,127,12,42949673684,Yahya Abdallahi,69,8,42949674571,Alim Guliyev,10,0

### Query 2
This query addresses the change of interests with large communities. It represents a version of query type 2 from the 2014 SIGMOD Programming contest. Unlike in the SIGMOD problem, the version for the DEBS Grand Challenge focuses on the dynamic change of query results over time, i.e., calls for a continuous evaluation of the results.

Goal of the query:
Given an integer k and a duration d (in seconds), find the k comments with the largest range, where the range of a comment is defined as the size of the largest connected component in the graph induced by persons who (a) have liked that comment (see likes, comments), (b) where the comment was created not more than d seconds ago, and (c) know each other (see friendships).

Parameters: k, d

Input Streams: likes, friendships, comments

Output:
The output includes a single timestamp ts and exactly k strings per line. The timestamp and the strings should be separated by commas. The k strings represent comments, ordered by range from largest to smallest, with ties broken by lexicographical ordering (ascending). The k strings and the corresponding timestamp must be printed only when some input triggers a change of the output, as defined above. If less than k strings can be determined, the character “-” (a minus sign without the quotations) should be printed in place of each missing string.

The field ts corresponds to the timestamp of the input data item that triggered an output update. For instance, a new friendship relation may change the size of a community with a shared interest and hence may change the k strings. The timestamp of the event denoting the added friendship relation is then the timestamp ts for that line's output. Also, the output must be updated when the results change due to the progress of time, e.g., when a comment is older that d. Specifically, if the update is triggered by an event leaving a time window at t2 (i.e., t2 = timestamp of the event + window size), the timestamp for the update is t2. As in Query 1, it is needless to say that timestamps refer to the logical time of the input data streams, rather than on the system clock.

In summary, the output is specified as follows:

ts: the timestamp of the tuple that triggers a change in the top-3 scoring active posts
comments_1,...,comment_k: top k comments ordered by range, starting with the largest range (comment_1).
Sample output tuples for the query with k=3 could look as follows:

2010-10-28T05:01:31.022+0000,I love strawberries,-,-
2010-10-28T05:01:31.024+0000,I love strawberries,what a day!,-
2010-10-28T05:01:31.027+0000,I love strawberries,what a day!,well done
2010-10-28T05:01:31.032+0000,what a day!,I love strawberries,well done

## Usage
Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.
Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat.
Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur.
Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.

## Authors
Giacomo Marciani [giacomo.marciani@gmail.com](mailto:giacomo.marciani@gmail.com)

Marco Piu [pyumarco@gmail.com](mailto:pyumarco@gmail.com)

Michele Porretta [micheleporretta@gmail.com](mailto:micheleporretta@gmail.com)

Matteo Nardelli [nardelli@ing.uniroma2.it](mailto:nardelli@ing.uniroma2.it)

Valeria Cardellini [cardellini@ing.uniroma2.it](mailto:cardellini@ing.uniroma2.it)

## Institution
Department of Civil Engineering and Computer Science Engineering, University of Rome Tor Vergata, Italy

## License
The paper is released under the [BSD License](https://opensource.org/licenses/BSD-3-Clause).
Please, read the file LICENSE.md for details.
