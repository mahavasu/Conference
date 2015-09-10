# Conference

About

This is a Cloud based API server engine project to support conference organization applictaction that exists
on the web. The API supports the following functionality
1.User authentication (via Google accounts)
2.User profiles
3.Conference information
4.Session information
5.User wishlists
6.The API is hosted on Google App Engine as application ID full-stack-conference, and can be accessed via the API explorer.

Setup Instructions

To deploy this API server locally, you should download and install the Google App Engine SDK for Python. 
Once installed, conduct the following steps:

Clone this repository. 
1.Update the value of application in app.yaml to the app ID you have registered in the App Engine admin console 
and would like to use to host your instance of this sample.
2.Update the values at the top of settings.py to reflect the respective client IDs you have registered in the [Developer Console][4].
3.Update the value of CLIENT_ID in static/js/app.js to the Web client ID
4.Run the app with the devserver using dev_appserver.py DIR, and ensure it's running by visiting your local server's address (by default [localhost:8080][5].)
5.Generate your client library(ies) with [the endpoints tool][6].
6.Deploy the application via appcfg.py update.


Google App Engine Python Docs
Programming Google App Engine (Book)
Data Modeling for Google App Engine Using Python and ndb (Screencast)
App Engine Modeling: Parent-Child Models (GitHub)
Design and Improvement Tasks

Task 1: Add Sessions to a Conference

I added the following endpoint methods:

createSession: creates a session, when given a conference.
getConferenceSessions:  returns all sessions ,  given a conference.
getConferenceSessionsByType:  returns all applicable sessions,given a conference and session type.
getSessionsBySpeaker:  returns all sessions across all conferences,given a speaker.

For the Session model design, following datastore properties:

Property	Type
name	string, required
highlights	string
speaker	string, required
duration	integer
typeOfSession	string, repeated
date	date
startTime	time
organizerUserId	string
In order to represent the one conference to many sessions relationship, I opted to use a parent-child implementation. This allows for strong consistent querying, as sessions can be queried by their conference ancestor. While this remits the possibility to move sessions between conferences, I reasoned that the trade-off in speed and consistency was worthwhile. People would want to know (i.e. query) about sessions quite often. Furthermore, sessions were Memcached to reflect that load.

In representing speakers, I contemplated linking the speaker field to user profiles. However, I decided against this, as it would force a speaker to have an account, to be registered. The obvious drawback here, though, is that querying by speaker could produce undesirable results with inconsistent entry, e.g. "John Bravo, Johnny Bravo, J. Bravo" would all be listed as separate speakers.

Session types (e.g. talk, lecture) were implemented more in a "tag" representation, with sessions able to receive multiple different types.

Task 2: Add Sessions to User Wishlist

I modified the Profile model to accommodate a 'wishlist' stored as a repeated key property field, named sessionsToAttend. In order to interact with this model in the API, I also had modify some of the previous methods in Task 1 to return a unique web-safe key for sessions. I added two endpoint methods to the API:

addSessionToWishlist: given a session websafe key, saves a session to a user's wishlist.
getSessionsInWishlist: return a user's wishlist.

Task 3: Indexes and Queries

I added two endpoint methods for additional queries that I thought would be useful for this application:

-getConferenceSessionFeed: returns a conference's sorted feed sessions occurring same day or later. This would be useful for users to see a chronologically sorted, upcoming "feed," similar to something like Meetup's feed. -getTBDSessions: returns sessions missing time/date information. Many times, conferences will know the speakers who will attend, but don't necessarily know the time and date that they will speak. As an administrative task, this might be a useful query to pair with some background methods to automatically notify the creator to fill in the necessary data as the conference or session dates approach.

For the specialized query, finding non-workshop sessions before 7pm, I ran into the limitations with using ndb/Datastore queries. Queries are only allowed to have one inequality filter, and it would cause a BadRequestError to filter on both startDate and typeOfSession. As a result, a workaround I implemented was to first query sessions before 7pm with ndb, and then manually filter that list with native Python to remove sessions with a 'workshop' type. This could have been done in reverse, and the query which would filter the most entities should be done with ndb.

Task 4: Add Featured Speaker

I modified the createSession endpoint to cross-check if the speaker appeared in any other of the conference's sessions. If so, the speaker name and relevant session names were added to the memcache under the featured_speaker key. I added a final endpoint, getFeaturedSpeaker, which would check the memcache for the featured speaker. If empty, it would simply pull the next upcoming speaker.


