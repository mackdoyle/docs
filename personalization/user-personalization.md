#Event-Based Recommendation Engine
###Collecting Data for User Recommendations Using Redis or Elasticsearch
This solution records meta information about a user and an event they trigger as a simple key/value store. Event triggers can be initiated by both anonymous and logged in users. 

#Requirements
- Collect data from anonymous and logged in users.
- Match existing event data from an anonymous user to their user profile upon user registration
- Store event data in Redis or ES. 
- Use a scalar value when storing data that implies relevance of the data being stored.
- User gets an engagement score. Points applied to different actions like sharing an article or viewing a video, reaching the end of a slideshow.

##Users
When a user enters the site, we immediately check for a cookie (or web storage) containing a user hash we may have stored during a previous visit. If we do not find one, we generate a hash to ID the user and store it  on the client. From this point forward, this will be used as an identifier. When the user triggers an event, their hash will be stored along with any event data in the recommendation engine.

When a user registers on the site, we will have already obtained or created a user hash. We can then use this to associate any data anonymously collected to a real person by either adding the user ID to all stored events in the recommendation engine or pull all data and storing in a secondary system, like a user profile engine.

##Event Collection
In Redis or ES, we can store objects. This allows us to collect as much information about a user triggered event and store it as a single record. The structure of the event record might look something like this:
```bash
Key: User ID <Hash>
Value: {
Event ID
Tag
Tag
Tag
Interest Type
Event Type
Location (The market associated with the event)
```

##Events Type
We should identify the type of event by defining Event Types that are stored with the event object when triggered. Each event type needs to have an arbitrary weight assigned to it for scoring relevance.
Initial Event Types are:
```bash
User login
Favoriting
Article Read
Persona
Tag(s)
```


##Relavance
Given an arbitrary event…
- Create a relevance score for each associated tag
- Create a relevance score for each user interested in a tag
- Create a (Re)score the user’s interest in a tag
 
Example:
User favorites an article tagged with `[coffee]` in the Chicago market. We would want to store any tags, interests types and the market name on that event. Each would have a different relevancy score with the market name receiving a higher score than a loosely related tag like 'trendy'

##User Interest Calculator
Compute the Uers’ interest in tags

##Redis vs Elasticsearch or other data store
Redis is very fast since it runs in memory. For that same reason, it may no scale well

##Technological approaches to explore
1. AR pushes event to Resque. Look into AR and Resque
2. Look into persistence and the headaches it may cause
3. Use of Duck typing
4. Batching with Redis Pipelines
5. Redis Lua scripting
6. Referential transparency when calculating

