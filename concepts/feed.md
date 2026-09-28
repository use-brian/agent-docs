---
title: Feed
description: Set your brand voice, plan content, refine drafts, review with your team, and deliver social posts or email campaigns.
canonical: https://usebrian.ai/docs/feed
tags: [feed, content, social, publishing, campaigns, email]
---

# Feed: plan, write, and publish

Feed is your workspace for content marketing. Keep ideas, your publishing plan, drafts, feedback, and results together, with Brian helping you write in your company's voice.

## Everyday workflow

1. **Set the voice and plan.** Describe your audience and tone, then turn ideas into planned posts. A calendar date alone does not publish.
2. **Write with Brian.** Create a post, add a private brief, and refine the draft. Review suggestions before accepting them.
3. **Review the destination.** Check the saved version and account before approving. Connected approval can publish; manual delivery needs a separate handoff.

## Open Feed and choose your channels

Open Feed from Home in your workspace. On your first visit, create a brand assistant and choose the channels you write for. You can start planning and drafting without connecting a social account.

Planning, voice, collaboration, and manual posting are available in both hosted and OSS Brian. Connected publishing, social insights, and inspiration depend on the platform and your deployment. See Platforms and editions below.

## Set your company and channel voice

Open Company → Voice to describe your audience, tone, preferred language, and things to avoid. Choose a channel to add its own voice rules. Company voice carries across channels; channel rules make the writing fit its destination. Review proposed voice changes before saving them.

## Turn ideas into a plan

Use Plan to capture an idea, set the month's goal and themes, and arrange content on the calendar. An idea can become a draft immediately or a planned slot. Open a slot to adjust its brief; choose Draft this to start writing from it.

Use Feed chat to ask for a plan or changes to an existing one. Review the proposed slots and add them individually or together. A date on the calendar organizes the work; it does not by itself authorize or schedule an external publication.

Example request: Plan three posts for next week about our new project. Use a practical tone and leave room for a customer question.

## Write and refine a post

Choose a channel and New post. Add a working title and a private brief, then write in the editor or ask Brian in Refine. The brief guides the conversation and is not published. Use Edit and Preview to check the content and its destination-specific presentation.

Select text to comment, suggest a change, or ask Brian about that passage. Review suggestions before accepting them, and use history and Undo when needed. Fill or remove unfinished text and image placeholders before confirming the draft. Generated content remains a candidate until you accept it.

## Review the saved version

Review checks the draft against the monthly plan, previous posts, its selected goal, relevant memory, and the content itself. Read both the findings and their coverage: missing sources or unavailable checks do not count as reviewed. Review is advisory; it does not edit, approve, or publish your post.

| Stage | What to do |
|---|---|
| Drafting | Write and refine, then choose Submit for approval to save a review version. |
| Needs review | Check the saved version. Save changes after editing, then use Approve when authorized. Connected approval can publish; check the destination before approving. |
| Ready | The approved content is waiting for manual delivery. It has not been published just because it is ready. |
| Posted | Delivery has been recorded. Check whether the receipt came from the provider or was confirmed manually. |

## Publish or complete the manual handoff

For a supported connected account, an owner or admin connects it in that channel's Settings. Check the account, final content, and any public-release confirmation before approving delivery. Follow the result shown in Feed; a timeout or an uncertain result is not permission to publish again.

1. For manual delivery, approve the draft into Ready and copy or export the final content.
2. Publish it in the destination app and check the actual result there.
3. Return to Feed, choose Mark as posted, and record the published link when available. This records your confirmation; it does not send the post.

## Platforms and editions

| Channel | Delivery path |
|---|---|
| Threads and X | Managed publishing for supported formats with a connected account. OSS can use an approved paid Feed Cloud Link. X supports a post or an ordered thread. |
| Instagram and XHS | Plan, draft, review, and copy for manual posting. Creating a draft does not connect an account or enable API publishing. |
| LinkedIn | Draft posts, link posts, or newsletter editions. Personal and company Page destinations are separate. Managed post delivery appears only when the deployment and account have the required capability; use the available status in Feed. |
| Email | Write and preview campaign emails. Sending requires an authorized SMTP sender, an approved audience and revision, and enabled delivery. |

LinkedIn newsletter editions use Prepare for LinkedIn and a manual editor handoff. Copy or export the content, upload the supplied images in LinkedIn, then record the actual edition URL. An export is not a publication receipt. Any follow-up link post is a separate draft and approval.

Feed Cloud Link lets an OSS workspace use paid hosted provider services while keeping its brain and drafts locally. A hosted owner or admin approves the link and its destination workspace. Authorized publishing content and media go to the hosted service; social account credentials stay there. Revoking the link stops managed access without removing the local manual workflow.

See [self-hosting](../self-hosting.md) for OSS installation.

## Campaigns, email, and results

Open Campaigns to group posts and emails around an objective. Add placements and tracked links to connect content with results. Changing a link's destination requires a new link and another review. Feed handles the content and results; CRM holds audiences, consent, and follow-up.

For email, check the subject, sender, reply-to, audience, personalization, and preview before approving a send or schedule. Approval belongs to that exact version and recipient set; later content changes require approval again. Pause or cancel affects future sends, and cannot recall accepted messages.

Read metrics by what they prove. Social insights require a supported connection; native campaign results require tracking to be configured. A click is not a verified conversion, and SMTP acceptance does not prove inbox delivery. Unavailable or delayed data is different from zero results.

See [campaign tracking and email](../api/campaigns.md) for the integration reference.

## Permissions, sync, and troubleshooting

- An action is unavailable: check your workspace and draft permissions. Connecting accounts and changing platform policies require an owner or admin; connection alone does not grant publishing authority.
- Changes have not synced: wait for the saved state before review or delivery. Resolve a conflict or preserve a separate copy instead of overwriting another person's edits.
- A draft cannot be approved: check unfinished placeholders, required fields, source access, and the saved revision. Access to a private source does not automatically authorize making it public.
- A delivery result is uncertain: check the provider or delivery receipt and reconcile it before retrying. A reconnect or retry must not create a duplicate post or email.
- Brian's generation or Review is unavailable: check model access, credits, and the displayed error. Ordinary planning and manual editing do not require a social connection.

## Related guides

- [Assistants](assistants.md)
- [CRM](crm.md)
- [Workspaces](workspaces.md)
- [Pricing and credits](../operations/pricing-and-credits.md)

