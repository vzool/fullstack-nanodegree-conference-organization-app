# Fullstack Nanodegree Conference Organization App

## Products
- [App Engine][1]

## Language
- [Python][2]

## APIs
- [Google Cloud Endpoints][3]

## Setup Instructions
1. Update the value of `application` in `app.yaml` to the app ID you
   have registered in the App Engine admin console and would like to use to host
   your instance of this sample.
1. Update the values at the top of `settings.py` to
   reflect the respective client IDs you have registered in the
   [Developer Console][4].
1. Update the value of CLIENT_ID in `static/js/app.js` to the Web client ID
1. (Optional) Mark the configuration files as unchanged as follows:
   `$ git update-index --assume-unchanged app.yaml settings.py static/js/app.js`
1. Run the app with the devserver using `dev_appserver.py DIR`, and ensure it's running by visiting your local server's address (by default [localhost:8080][5].)
1. (Optional) Generate your client library(ies) with [the endpoints tool][6].
1. Deploy your application.

# Solution

To test all endpoints API you need to run `dev_appserver.py <DIR>` and then open this URL at your favorite browser:

`http://localhost:8080/_ah/api/explorer`

Remember, it's `localhost` not `127.0.0.1` which it doesn't work for me.

## Task #1

I have created Session Model as `ndb.Model` and SessionForm as `messages.Message` with the following attributes:

- Session name
- highlights
- speaker
- duration
- typeOfSession
- date
- start time (in 24 hour notation so it can be ordered).

Attributes looking after normalization in `ndb.Model` and `messages.Message`:

```python

class Session(ndb.Model):
    """Session -- session object"""
    name = ndb.StringProperty()
    highlights = ndb.StringProperty(repeated=True)
    speaker = ndb.StringProperty()
    duration = ndb.IntegerProperty()
    typeOfSession = ndb.StringProperty(default='NOT_SPECIFIED')
    date = ndb.DateProperty()
    startTime = ndb.IntegerProperty()

class SessionForm(messages.Message):
    """Session Form -- Session outbound from message"""
    name = messages.StringField(1)
    highlights = messages.StringField(2, repeated=True)
    speaker = messages.StringField(3)
    duration = messages.IntegerField(4)
    typeOfSession = messages.EnumField('SessionType', 5)
    date = messages.StringField(6)
    startTime = messages.IntegerField(7)
    websafeKey = messages.StringField(8)

```

### How did I add session to conference?

I put speaker inside Session Model, because every session may have different speaker about the same conference.

So, I added a new attribute in Conference `ndb.Model` which is `sessionKeysScheduleList` and it's an array of all sessions that conference have, and `sessionKeysScheduleList` is a `ndb.StringProperty(repeated=True)`

If there is any new session at specific conference, that session `safeurl` will be save at `sessionKeysScheduleList`.

If any session deleted it should be deleted from `sessionKeysScheduleList` too, that's will make `sessionKeysScheduleList` up to date with all exist sessions.

Modifications that take place at `Conference` objects:

```python
class Conference(ndb.Model):
    # ..
    # ..
    sessionKeysScheduleList     = ndb.StringProperty(repeated=True)

class ConferenceForm(messages.Message):
    # ..
    # ..
    sessionKeysScheduleList = messages.StringField(13,repeated=True)
```

Conference Session has many endpoints which are they:

- getConferenceSessions(websafeConferenceKey): get all Sessions for specific Conference.
- getConferenceSessionsByType(websafeConferenceKey, typeOfSession): get all Sessions with specific type(Lecture, Workshop, ..., etc).
- getSessionsBySpeaker(speaker): get all Sessions by speaker.
- createSession(SessionForm, websafeConferenceKey): create new Session for specific Conference.

### How Query Works?

In fact when I like to generate new Session, Google Datastore method is create Key from Profile Key based on user ID and Session ID based on Profile key.

So, Session Key will like this:

```yaml
User ID -> Profile ID -> Session ID
```

Which leads to Entity in Session Kind.

## Task #2

To implement wishlist session for a user, I modified `Profile` ndb.Model by adding a new attribute that can save all wishlist sessions, and that's attribute is `Wishlist` which is a `ndb.StringProperty(repeated=True)`.

Yes, `Wishlist` is an array for All user sessions that he/she wish it.
Of course, if and deletion for any session that's will lead to delete session key from `Wishlist` too.


Modifications that take place at `Profile` objects:

```python
class Profile(ndb.Model):
    # ..
    # ..
    Wishlist = ndb.StringProperty(repeated=True)

class ProfileForm(messages.Message):
    # ..
    # ..
    Wishlist = messages.StringField(5, repeated=True)
```

Wishlist Session has two endpoints which are:

- getSessionsInWishlist(SessionKey): get all Sessions added to wishlist.
- addSessionToWishlist(): add new session to wishlist.


## Task #3

The main problem that I faced with Queries in General was I don't know how to query Multiple Entities by `urlsafe` and then filter them with specific filters in two steps only.

My methodology to solve this problem as following:

- Convert `urlsafe` into `ndb.Key`.
- Append all `ndb.Keys`.
- Finally, query on `ndb.Keys` and then return `items` which are `Session` Kind to rest of the endpoint Function.

Always, Don't help Google by making things that already done for you, Let Google help you if you believe in them.


```python
# Loop through session keys and get session data itself
sess_keys = [ndb.Key(urlsafe=wsck) for wsck in conf.sessionKeysScheduleList]
items = ndb.get_multi(sess_keys)
```


## Task #4

I implemented Featured Speaker by made a static function inside `ConferenceApi` which will be accessible from `main.py`.

I made `_cacheSessionAnnouncement` and I cache Session query result into Memcache, even you can access this feature by this url: `http://localhost:8080/speaker/featured` and if you want to check the result from cached data check this url: `http://apis-explorer.appspot.com/apis-explorer/?base=http://localhost:8080/_ah/api#p/conference/v1/conference.getSessionAnnouncement?_h=60&`.

To make this work for me I made an `GetFeaturedSpeakerHandler` handler for that request in `main.py` script file.

```python
# Class Handler
class GetFeaturedSpeakerHandler(webapp2.RequestHandler):
    def get(self):
        """Gety featured speaker request"""
        ConferenceApi._getFeaturedSpeaker()
```

Add additional line in `app = webapp2.WSGIApplication`, and it look like this:

```python
app = webapp2.WSGIApplication([
    # ..
    ('/speaker/featured', GetFeaturedSpeakerHandler),
], debug=True)

``` 

## Additional Task

I worked with querys, and I made it.

The question was How to make a Query about all session not Workshop and before 7 pm?

So, there is `NOT EQUAL` and `LESS THAT` tokens in the same filter, which it is not acceptable by Google.
And always an Error message appears said: "No not possible".

After I dig in the internet I found ComputedProperty, which are conditions in the Model itself, strange and weird.
And as you can see below there is a condition in way of `ComputedProperty`.

```python

class Session(ndb.Model):
   
    # ..

    session_condition_1 = ndb.ComputedProperty(lambda self: self.typeOfSession != 'WORKSHOP' and self.startTime < 19)

```

When you like to trigger that condition at any Entity inside Session Kind you can just do this:

```python
q = Session.query(
    Session.session_condition_1 == True
)
        
q.get()

```

Finally, Google never complains again.


# Licence

It's Completely Free. But, Do whatever you like to do on your own full responsibility;

This licence is known with [MIT License][7] in professional networks.

[1]: https://developers.google.com/appengine
[2]: http://python.org
[3]: https://developers.google.com/appengine/docs/python/endpoints/
[4]: https://console.developers.google.com/
[5]: https://localhost:8080/
[6]: https://developers.google.com/appengine/docs/python/endpoints/endpoints_tool
[7]: http://vzool.mit-license.org