# NewsFeed system 

<!-- MarkdownTOC -->

- [Scenario](#scenario)
	- [Post a tweet](#post-a-tweet)
	- [Timeline](#timeline)
	- [News feed](#news-feed)
	- [Follow / Unfollow a user](#follow--unfollow-a-user)
	- [Register / Login](#register--login)
- [Service](#service)
	- [User service](#user-service)
	- [Tweet service](#tweet-service)
	- [Media service](#media-service)
	- [Friendship service](#friendship-service)
- [Storage](#storage)
	- [Storage mechanism](#storage-mechanism)
		- [SQL database](#sql-database)
		- [NoSQL database](#nosql-database)
		- [File system](#file-system)
	- [Schema design](#schema-design)
		- [User table](#user-table)
		- [Friendship table](#friendship-table)
		- [Tweet table](#tweet-table)
	- [Architecture](#architecture)
		- [Pull model](#pull-model)
		- [Push model](#push-model)
		- [Push vs Pull](#push-vs-pull)
			- [Push use case](#push-use-case)
			- [Pull use case](#pull-use-case)
- [Scale](#scale)
	- [Push combine with pull](#push-combine-with-pull)
		- [Oscillation related problem](#oscillation-related-problem)
	- [Optimize pull](#optimize-pull)
	- [Optimize push](#optimize-push)
	- [Hot spot / Thundering herd problem](#hot-spot--thundering-herd-problem)
- [Follow up](#follow-up)
	- [Follow and unfollow](#follow-and-unfollow)
	- [How to store likes](#how-to-store-likes)

<!-- /MarkdownTOC -->


## Scenario
### Post a tweet
### Timeline
### News feed
### Follow / Unfollow a user
### Register / Login

## Service
### User service
* Register
* Login

### Tweet service
* Post a tweet
* Newsfeed
* Timeline

### Media service
* Upload image
* Upload video

### Friendship service
* Follow
* Unfollow

## Storage

### Storage mechanism
#### SQL database
* User table

#### NoSQL database
* Tweets
* Social graph (followers)

#### File system
* Images
* Media files

### Schema design
#### User table
* id: integer
* username: varchar
* email: varchar
* password: varchar

#### Friendship table
* id: integer
* from_user_id: foreign key
* to_user_id: foreign key

#### Tweet table
* id: integer
* user_id: foreign key
* content: text
* created_at: timestamp

### Architecture
#### Pull model
* Algorithm

```
// each following's first 100 tweets, merge with a key way sort

getNewsFeed(request)
	followings = DB.getFollowings(user=request.user)
	newsFeed = empty
	for follow in followings:
		tweets = DB.getTweets(follow.toUser, 100)
		newsFeed.merge(tweets)
	sort(newsFeeds)
	return newsFeed

postTweet(request, tweet)
	DB.insertTweet(request.User, tweet)
	return success
```

* Complexity
	- Get news feed: N DB reads + K way merge
	- Post a tweet: 1 DB write

#### Push model
* Algorithm

```
// Each time after a user tweet, fanout his tweets to all followers' feed list

getNewsFeed(request)
	return DB.getNewsFeed(request.user)

postTweet(request, tweetInfo)
	tweet = DB.insertTweet(request.user, tweetInfo)
	AsyncService.fanoutTweet(request.user, tweet)
	return success

AsyncService::fanoutTweet(user, tweet)
	followers = DB.getFollowers(user)
	for follower in followers:
		DB.insertNewsFeed(tweet, follower)
```

* For each user, establish a List to store his news feed info
	- NewsFeed table
		+ id	integer
		+ ownerId	foreign key
		+ tweetId	foreign key
		+ createdAt	timestamp

* Complexity:
	- Get news feed: 1 DB read
	- Post a tweet: N followers, N DB writes. Executed asynchronously. 

#### Push vs Pull
##### Push use case
* Not restrict on latency
* Bi-direction relationship

##### Pull use case
* Low latency
* Single direction relationship

## Scale 
### Push combine with pull
#### Oscillation related problem
* Threshold (number of followers)
	- Below threshold use push
	- Above threshold use pull
* For popular users, do not push. Followers fetch from their timeline and integrate into news feed. 

### Optimize pull
* Add cache
* Cache each user's timeline
* Cache each user's newsFeed

### Optimize push
* For inactive users, do not push
	- Rank followers by weight

### Hot spot / Thundering herd problem
* Cache

## Follow up
### Follow and unfollow
* Asynchronously executed
	- Merge timeline into news feed asynchronously
	- Pick out tweets from news feed asynchronously

### How to store likes
* Tweet table

| Columns     | Type        | 
|-------------|-------------| 
| id          | integer     | 
| userId      | foreign key | 
| content     | text        | 
| createdAt   | timestamp   | 
| likeNums    | integer     | 
| commentNums | integer     | 
| retweetNums | integer     | 

* Like table

| Columns   | Type       | 
|-----------|------------| 
| id        | integer    | 
| userId    | foreignKey | 
| tweetId   | foreignKey | 
| createdAt | timestamp  | 

