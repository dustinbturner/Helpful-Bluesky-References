# Quick Start Guide to Building Applications on AT Protocol

[Find the source code for the example application on GitHub]

In this guide, we're going to build a simple multi-user app that publishes your current "status" as an emoji. Our application will be a status dashboard with user authentication and real-time updates.

We will cover how to:
- Sign in via OAuth
- Fetch information about users (profiles)
- Listen to the network firehose for new data
- Publish data on the user's account using a custom schema

We're going to keep this light so you can quickly wrap your head around ATProto. There will be links with more information about each step.

## Introduction

Data in the Atmosphere is stored on users' personal repos. It's almost like each user has their own website. Our goal is to aggregate data from the users into our SQLite DB.

Think of our app like a Google. If Google's job was to say which emoji each website had under /status.json, then it would show something like:

- nytimes.com is feeling 📰 according to https://nytimes.com/status.json
- bsky.app is feeling 🦋 according to https://bsky.app/status.json
- reddit.com is feeling 🤓 according to https://reddit.com/status.json

The Atmosphere works the same way, except we're going to check `at://` instead of `https://`. Each user has a data repo under an `at://` URL. We'll crawl all the user data repos in the Atmosphere for all the "status.json" records and aggregate them into our SQLite database.

`at://` is the URL scheme of the AT Protocol. Under the hood it uses common tech like HTTP and DNS, but it adds all of the features we'll be using in this tutorial.

## Step 1. Starting with our ExpressJS app

Start by cloning the repo and installing packages:

```bash
git clone https://github.com/bluesky-social/statusphere-example-app.git
cd statusphere-example-app
cp .env.template .env
npm install
npm run dev
# Navigate to http://localhost:8080
```

Our repo is a regular Web app. We're rendering our HTML server-side like it's 1999. We also have a SQLite database that we're managing with Kysely.

Our starting stack:
- Typescript
- NodeJS web server (express)
- SQLite database (Kysely)
- Server-side rendering (uhtml)

## Step 2. Signing in with OAuth

When somebody logs into our app, they'll give us read & write access to their personal at:// repo. We'll use that to write the status json record.

We're going to accomplish this using OAuth (spec). Most of the OAuth flows are going to be handled for us using the @atproto/oauth-client-node library.

Our login page just asks the user for their "handle," which is the domain name associated with their account. For Bluesky users, these tend to look like alice.bsky.social, but they can be any kind of domain (eg alice.com).

```html
<!-- src/pages/login.ts -->
<form action="/login" method="post" class="login-form">
  <input
    type="text"
    name="handle"
    placeholder="Enter your handle (eg alice.bsky.social)"
    required
  />
  <button type="submit">Log in</button>
</form>
```

When they submit the form, we tell our OAuth client to initiate the authorization flow:

```typescript
/** src/routes.ts **/
// Login handler
router.post(
  '/login',
  handler(async (req, res) => {
    // Initiate the OAuth flow
    const handle = req.body?.handle
    const url = await oauthClient.authorize(handle, {
      scope: 'atproto transition:generic',
    })
    return res.redirect(url.toString())
  })
)
```

When that finishes, the user will be sent back to /oauth/callback on our Web app:

```typescript
/** src/routes.ts **/
// OAuth callback to complete session creation
router.get(
  '/oauth/callback',
  handler(async (req, res) => {
    // Store the credentials
    const { session } = await oauthClient.callback(params)

    // Attach the account DID to our user via a cookie
    const cookieSession = await getIronSession(req, res)
    cookieSession.did = session.did
    await cookieSession.save()

    // Send them back to the app
    return res.redirect('/')
  })
)
```

## Step 3. Fetching the user's profile

In Bluesky, users publish a "profile" record which looks like this:

```typescript
interface ProfileRecord {
  displayName?: string // a human friendly name
  description?: string // a short bio
  avatar?: BlobRef     // small profile picture
  banner?: BlobRef     // banner image to put on profiles
  createdAt?: string   // declared time this profile data was added
  // ...
}
```

You can examine this record directly using atproto-browser.vercel.app. We're going to use the Agent associated with the user's OAuth session to fetch this record:

```typescript
await agent.com.atproto.repo.getRecord({
  repo: agent.assertDid,                // The user
  collection: 'app.bsky.actor.profile', // The collection
  rkey: 'self',                         // The record key
})
```

Let's update our homepage to fetch this profile record:

```typescript
/** src/routes.ts **/
// Homepage
router.get(
  '/',
  handler(async (req, res) => {
    // If the user is signed in, get an agent which communicates with their server
    const agent = await getSessionAgent(req, res, ctx)

    if (!agent) {
      // Serve the logged-out view
      return res.type('html').send(page(home()))
    }

    // Fetch additional information about the logged-in user
    const { data: profileRecord } = await agent.com.atproto.repo.getRecord({
      repo: agent.assertDid,                // our user's repo
      collection: 'app.bsky.actor.profile', // the bluesky profile record type
      rkey: 'self',                         // the record's key
    })

    // Serve the logged-in view
    return res
      .type('html')
      .send(page(home({ profile: profileRecord.value || {} })))
  })
)
```

## Step 4. Reading & writing records

You can think of the user repositories as collections of JSON records. Let's look at how we write "status" records:

```typescript
// Generate a time-based key for our record
const rkey = TID.nextStr()

// Write the record
await agent.com.atproto.repo.putRecord({
  repo: agent.assertDid,                 // The user
  collection: 'xyz.statusphere.status',  // The collection
  rkey,                                 // The record key
  record: {                             // The record value
    status: "👍",
    createdAt: new Date().toISOString()
  }
})
```

Our POST /status route:

```typescript
/** src/routes.ts **/
// "Set status" handler
router.post(
  '/status',
  handler(async (req, res) => {
    // If the user is signed in, get an agent which communicates with their server
    const agent = await getSessionAgent(req, res, ctx)
    if (!agent) {
      return res.status(401).type('html').send('<h1>Error: Session required</h1>')
    }

    // Construct their status record
    const record = {
      $type: 'xyz.statusphere.status',
      status: req.body?.status,
      createdAt: new Date().toISOString(),
    }

    try {
      // Write the status record to the user's repository
      await agent.com.atproto.putRecord({
        repo: agent.assertDid, 
        collection: 'xyz.statusphere.status',
        rkey: TID.nextStr(),
        record,
      })
    } catch (err) {
      logger.warn({ err }, 'failed to write record')
      return res.status(500).type('html').send('<h1>Error: Failed to write record</h1>')
    }

    res.status(200).json({})
  })
)
```

## Step 5. Creating a custom "status" schema

Repo collections are typed, meaning that they have a defined schema. The app.bsky.actor.profile type definition can be found in the documentation.

Anybody can create a new schema using the Lexicon language, which is very similar to JSON-Schema. The schemas use reverse-DNS IDs which indicate ownership.

Let's create our schema:

```json
/** lexicons/status.json **/
{
  "lexicon": 1,
  "id": "xyz.statusphere.status",
  "defs": {
    "main": {
      "type": "record",
      "key": "tid",
      "record": {
        "type": "object",
        "required": ["status", "createdAt"],
        "properties": {
          "status": {
            "type": "string",
            "minLength": 1,
            "maxGraphemes": 1,
            "maxLength": 32
          },
          "createdAt": {
            "type": "string",
            "format": "datetime"
          }
        }
      }
    }
  }
}
```

Now let's run code-generation using our schema:

```bash
./node_modules/.bin/lex gen-server ./src/lexicon ./lexicons/*
```

## Step 6. Listening to the firehose

Using a Relay service we can listen to an aggregated firehose of events across all users in the network:

```typescript
/** src/ingester.ts **/
import { Firehose } from '@atproto/sync'
import * as Status from '#/lexicon/types/xyz/statusphere/status'

new Firehose({
  filterCollections: ['xyz.statusphere.status'],
  handleEvent: async (evt) => {
    // Watch for write events
    if (evt.event === 'create' || evt.event === 'update') {
      const record = evt.record

      // If the write is a valid status update
      if (
        evt.collection === 'xyz.statusphere.status' &&
        Status.isRecord(record) &&
        Status.validateRecord(record).success
      ) {
        // Store the status in our SQLite
        await db
          .insertInto('status')
          .values({
            uri: evt.uri.toString(),
            authorDid: evt.author,
            status: record.status,
            createdAt: record.createdAt,
            indexedAt: new Date().toISOString(),
          })
          .onConflict((oc) =>
            oc.column('uri').doUpdateSet({
              status: record.status,
              indexedAt: new Date().toISOString(),
            })
          )
          .execute()
      }
    }
  },
})
```

## Step 7. Listing the latest statuses

Now we can produce a timeline of status updates:

```typescript
/** src/routes.ts **/
// Homepage
router.get(
  '/',
  handler(async (req, res) => {
    // Fetch data stored in our SQLite
    const statuses = await db
      .selectFrom('status')
      .selectAll()
      .orderBy('indexedAt', 'desc')
      .limit(10)
      .execute()

    // Map user DIDs to their domain-name handles
    const didHandleMap = await resolver.resolveDidsToHandles(
      statuses.map((s) => s.authorDid)
    )
  })
)
```

## Step 8. Optimistic updates

As a final optimization, let's introduce "optimistic updates" by updating our SQLite DB immediately after writing to the user's repo:

```typescript
/** src/routes.ts **/
// "Set status" handler
router.post(
  '/status',
  handler(async (req, res) => {
    // Write to repo...
    
    try {
      // Optimistically update our SQLite
      await db
        .insertInto('status')
        .values({
          uri,
          authorDid: agent.assertDid, 
          status: record.status,
          createdAt: record.createdAt,
          indexedAt: new Date().toISOString(),
        })
        .execute()
    } catch (err) {
      logger.warn(
        { err },
        'failed to update computed view; ignoring as it should be caught by the firehose'
      )
    }
  })
)
```

## Thinking in AT Proto

When building your app, think in these four key steps:

1. Design the Lexicon schemas for the records you'll publish into the Atmosphere
2. Create a database for aggregating the records into useful views
3. Build your application to write the records on your users' repos
4. Listen to the firehose to aggregate data across the network

## Next steps

If you want to practice what you've learned, here are some additional challenges you could try:

- Sync the profile records of all users so that you can show their display names instead of their handles
- Count the number of each status used and display the total counts
- Fetch the authed user's app.bsky.graph.follow follows and show statuses from them
- Create a different kind of schema, like a way to post links to websites and rate them 1 through 4 stars
