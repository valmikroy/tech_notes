# Designing Twitter



Why?

- To let people communicate to the masses in least number of words. 



How?

- People should be able to post a tweet
- Read tweet 
- Tweet can contain Text, photos, videos 



What?

- Latency requirements?
  - How quickly to distribute to the users
- Storage requirements?
  - Tweetin freq





- 1M users
- Average 10 tweets per day
- 100KB per tweet
- Average followers 100 



1M x 10 x 100KB = 1TB storage per day 

1M x 10 x 100KB = 1GB per day delivery for 10M querys per day  ~= 115 queries/second



What Missed?

- Daily Active Users

- Data sharding?

- Sharding by user or by Tweet?  

- Shard by user and cache most recent tweets.

- Write through cache for tweets?

- To fetch tweet, you go through list of followers, fetch their last timestamp locally when tweets got fetched, query servers to get newest tweet then sort them with time and present.

  











 

















