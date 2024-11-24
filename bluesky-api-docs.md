# Bluesky and AT Protocol API Reference

## Introduction
This document contains HTTP API reference documentation for Bluesky and AT Protocol lexicons. Public endpoints which don't require authentication can be made directly against the public Bluesky AppView API: `https://public.api.bsky.app`. Authenticated requests are usually made to the user's PDS, with automatic service proxying.

## API Categories

### Bluesky Application APIs (app.bsky.*)

#### Actor Management
- `app.bsky.actor.getPreferences`
- `app.bsky.actor.getProfile`
- `app.bsky.actor.getProfiles`
- `app.bsky.actor.getSuggestions`
- `app.bsky.actor.putPreferences`
- `app.bsky.actor.searchActorsTypeahead`
- `app.bsky.actor.searchActors`

#### Feed Management
- `app.bsky.feed.describeFeedGenerator`
- `app.bsky.feed.getActorFeeds`
- `app.bsky.feed.getActorLikes`
- `app.bsky.feed.getAuthorFeed`
- `app.bsky.feed.getFeedGenerator`
- `app.bsky.feed.getFeedGenerators`
- `app.bsky.feed.getFeedSkeleton`
- `app.bsky.feed.getFeed`
- `app.bsky.feed.getLikes`
- `app.bsky.feed.getListFeed`
- `app.bsky.feed.getPostThread`
- `app.bsky.feed.getPosts`
- `app.bsky.feed.getQuotes`
- `app.bsky.feed.getRepostedBy`
- `app.bsky.feed.getSuggestedFeeds`
- `app.bsky.feed.getTimeline`
- `app.bsky.feed.searchPosts`
- `app.bsky.feed.sendInteractions`

#### Graph Operations
- `app.bsky.graph.getActorStarterPacks`
- `app.bsky.graph.getBlocks`
- `app.bsky.graph.getFollowers`
- `app.bsky.graph.getFollows`
- `app.bsky.graph.getKnownFollowers`
- `app.bsky.graph.getListBlocks`
- `app.bsky.graph.getListMutes`
- `app.bsky.graph.getList`
- `app.bsky.graph.getLists`
- `app.bsky.graph.getMutes`
- `app.bsky.graph.getRelationships`
- `app.bsky.graph.getStarterPack`
- `app.bsky.graph.getStarterPacks`
- `app.bsky.graph.getSuggestedFollowsByActor`
- `app.bsky.graph.muteActorList`
- `app.bsky.graph.muteActor`
- `app.bsky.graph.muteThread`
- `app.bsky.graph.unmuteActorList`
- `app.bsky.graph.unmuteActor`
- `app.bsky.graph.unmuteThread`

#### Notifications
- `app.bsky.notification.getUnreadCount`
- `app.bsky.notification.listNotifications`
- `app.bsky.notification.putPreferences`
- `app.bsky.notification.registerPush`
- `app.bsky.notification.updateSeen`

#### Labeler
- `app.bsky.labeler.getServices`

#### Video
- `app.bsky.video.getJobStatus`
- `app.bsky.video.getUploadLimits`
- `app.bsky.video.uploadVideo`

### Bluesky Chat APIs (chat.bsky.*)
All chat endpoints require authentication and are proxied to the central chat service (`did:web:api.bsky.chat`).

#### Account Management
- `chat.bsky.actor.deleteAccount`
- `chat.bsky.actor.exportAccountData`

#### Conversation Management
- `chat.bsky.convo.deleteMessageForSelf`
- `chat.bsky.convo.getConvoForMembers`
- `chat.bsky.convo.getConvo`
- `chat.bsky.convo.getLog`
- `chat.bsky.convo.getMessages`
- `chat.bsky.convo.leaveConvo`
- `chat.bsky.convo.listConvos`
- `chat.bsky.convo.muteConvo`
- `chat.bsky.convo.sendMessageBatch`
- `chat.bsky.convo.sendMessage`
- `chat.bsky.convo.unmuteConvo`
- `chat.bsky.convo.updateRead`

#### Moderation
- `chat.bsky.moderation.getActorMetadata`
- `chat.bsky.moderation.getMessageContext`
- `chat.bsky.moderation.updateActorAccess`

### AT Protocol Admin APIs (com.atproto.admin.*)
These endpoints require admin authentication and are made directly to the PDS instance.

- `com.atproto.admin.deleteAccount`
- `com.atproto.admin.disableAccountInvites`
- `com.atproto.admin.disableInviteCodes`
- `com.atproto.admin.enableAccountInvites`
- `com.atproto.admin.getAccountInfo`
- `com.atproto.admin.getAccountInfos`
- `com.atproto.admin.getInviteCodes`
- `com.atproto.admin.getSubjectStatus`
- `com.atproto.admin.searchAccounts`
- `com.atproto.admin.sendEmail`
- `com.atproto.admin.updateAccountEmail`
- `com.atproto.admin.updateAccountHandle`
- `com.atproto.admin.updateAccountPassword`
- `com.atproto.admin.updateSubjectStatus`

### AT Protocol Core APIs (com.atproto.*)

#### Identity Management
- `com.atproto.identity.getRecommendedDidCredentials`
- `com.atproto.identity.requestPlcOperationSignature`
- `com.atproto.identity.resolveHandle`
- `com.atproto.identity.signPlcOperation`
- `com.atproto.identity.submitPlcOperation`
- `com.atproto.identity.updateHandle`

#### Repository Management
- `com.atproto.repo.applyWrites`
- `com.atproto.repo.createRecord`
- `com.atproto.repo.deleteRecord`
- `com.atproto.repo.describeRepo`
- `com.atproto.repo.getRecord`
- `com.atproto.repo.importRepo`
- `com.atproto.repo.listMissingBlobs`
- `com.atproto.repo.listRecords`
- `com.atproto.repo.putRecord`
- `com.atproto.repo.uploadBlob`

#### Server Management
- `com.atproto.server.activateAccount`
- `com.atproto.server.checkAccountStatus`
- `com.atproto.server.confirmEmail`
- `com.atproto.server.createAccount`
- `com.atproto.server.createAppPassword`
- `com.atproto.server.createInviteCode`
- `com.atproto.server.createInviteCodes`
- `com.atproto.server.createSession`
- `com.atproto.server.deactivateAccount`
- `com.atproto.server.deleteAccount`
- `com.atproto.server.deleteSession`
- `com.atproto.server.describeServer`
- `com.atproto.server.getAccountInviteCodes`
- `com.atproto.server.getServiceAuth`
- `com.atproto.server.getSession`
- `com.atproto.server.listAppPasswords`
- `com.atproto.server.refreshSession`
- `com.atproto.server.requestAccountDelete`
- `com.atproto.server.requestEmailConfirmation`
- `com.atproto.server.requestEmailUpdate`
- `com.atproto.server.requestPasswordReset`
- `com.atproto.server.reserveSigningKey`
- `com.atproto.server.resetPassword`
- `com.atproto.server.revokeAppPassword`
- `com.atproto.server.updateEmail`

#### Sync
- `com.atproto.sync.getBlob`
- `com.atproto.sync.getBlocks`
- `com.atproto.sync.getLatestCommit`
- `com.atproto.sync.getRecord`
- `com.atproto.sync.getRepoStatus`
- `com.atproto.sync.getRepo`
- `com.atproto.sync.listBlobs`
- `com.atproto.sync.listRepos`
- `com.atproto.sync.notifyOfUpdate`
- `com.atproto.sync.requestCrawl`

### Ozone Moderation Service APIs (tools.ozone.*)
These endpoints require authentication and are proxied to the Ozone instance.

#### Communication
- `tools.ozone.communication.createTemplate`
- `tools.ozone.communication.deleteTemplate`
- `tools.ozone.communication.listTemplates`
- `tools.ozone.communication.updateTemplate`

#### Moderation
- `tools.ozone.moderation.emitEvent`
- `tools.ozone.moderation.getEvent`
- `tools.ozone.moderation.getRecord`
- `tools.ozone.moderation.getRecords`
- `tools.ozone.moderation.getRepo`
- `tools.ozone.moderation.getRepos`
- `tools.ozone.moderation.queryEvents`
- `tools.ozone.moderation.queryStatuses`
- `tools.ozone.moderation.searchRepos`

#### Server Management
- `tools.ozone.server.getConfig`

#### Set Operations
- `tools.ozone.set.addValues`
- `tools.ozone.set.deleteSet`
- `tools.ozone.set.deleteValues`
- `tools.ozone.set.getValues`
- `tools.ozone.set.querySets`
- `tools.ozone.set.upsertSet`

#### Signature Analysis
- `tools.ozone.signature.findCorrelation`
- `tools.ozone.signature.findRelatedAccounts`
- `tools.ozone.signature.searchAccounts`

#### Team Management
- `tools.ozone.team.addMember`
- `tools.ozone.team.deleteMember`
- `tools.ozone.team.listMembers`
- `tools.ozone.team.updateMember`
