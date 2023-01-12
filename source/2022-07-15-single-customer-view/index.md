---
title: Single Customer View
tags: Genesys Cloud, Developer Engagement
date: 2022-07-15
author: andrew.johnson
category: 7
---

Single Customer View is an upcoming feature of Genesys Cloud that gives agents a view of all the interactions that a customer has had with your organization. A primary goal of this feature is to associate all interactions with Contacts. This facilitates building a single view of the customer so that Genesys Cloud agents can easily answer the questions: “Who am I talking to?” and “How have they interacted with my organization in the past?” 

Several new concepts are being added to the platform API in order to support this new feature and we will explore them in this blog post together. Keep in mind that these new concepts should be non-disruptive to existing organizations; you will not be required to engage with most of these changes until your organization starts to use some of the new features described below.

## Identifiers

Until now, the way to look up contacts by identifiers like phone number, email address, etc. has been to use the [`GET /api/v2/externalcontacts/reversewhitepageslookup`](/commdigital/externalcontacts/externalcontacts-apis#get-api-v2-externalcontacts-reversewhitepageslookup) endpoint. This endpoint returns a list of contacts that contain an identifier with the provided lookup string.

With the introduction of identity resolution functionality to Genesys Cloud, External Contacts introduces a new type of object called an Identifier.  An identifier has a type and a value and points to exactly one contact.  When the platform resolves an identifier for an interaction to a contact, it looks up its identifier and returns the contact it points to.  If no contact yet exists, it creates the contact and points the identifier to it.  We say that it has claimed the identifier for that contact.

When contacts are created using the [`POST /api/v2/externalcontacts/contacts`](/commdigital/externalcontacts/externalcontacts-apis#post-api-v2-externalcontacts-contacts) endpoint, the platform attempts to claim Identifiers that correspond to each of the named identifier fields (such as cellPhone, workEmail, etc.).  If any identifier is already claimed by another contact, it remains claimed by that contact. As contacts are edited via the [`PUT`](/commdigital/externalcontacts/externalcontacts-apis#put-api-v2-externalcontacts-contacts--contactId-) and [`DELETE`](/commdigital/externalcontacts/externalcontacts-apis#delete-api-v2-externalcontacts-contacts--contactId-) endpoints, the identifiers corresponding to the contact are claimed/released appropriately. 

You can read more about identifiers and their associated operations in our [developer documentation](/commdigital/externalcontacts/contact-merges#identifiers).

## Contact Types

When the platform receives a new interaction on a supported channel, it resolves the identifiers for that interaction (for example: phone number, email address, social media identifier, etc.) to a contact.  If no matching contact is found, it creates one automatically.  It then sets the `participant.externalContactId` field on the [converesation object](/routing/conversations/conversations-apis#get-api-v2-conversations--conversationId-) to the ID of the found/created contact.

These autogenerated contacts need to be distinguished from the contacts that your organization creates explicitly. To do this, we are introducing three types of contacts and a new `type` field on the Contact object.

* **Curated** contacts are the contacts you already know (and hopefully love). These are created via the API, the web application/desktop app, or via CSV/Salesforce imports. They never age out; you are responsible for their full lifecycle and eventual deletion just as you have always been.
* **Identified** contacts are a new kind of automatically generated contact. These contacts contain some kind of PII and will automatically age out of the system to help platform customers maintain compliance with data privacy laws and follow the data-minimization principle.
* **Ephemeral** contacts are also a kind of automatically generated contact. However, these contacts contain no PII; they are created when only there are no identifiers or just a cookie associated to a customer interaction. Just like identified contacts, these will automatically age out of the system.

Neither kind of autogenerated contact appears in search results, list/scan endpoints, or in reverse whitepages lookups. This is done to keep them largely out of the way for existing users. These contacts will currently appear only when queried for directly.

Autogenerated contacts may be promoted to curated contacts to keep them in the curated set permanently. You may do this using the new [`POST /api/v2/externalcontacts/contacts/{contactId}/promotion`](/platform/preview-apis#post-api-v2-externalcontacts-contacts--contactId--promotion) API endpoint or the appropriate buttons in the Genesys Cloud web application. When you do so, the promoted contact will appear in all the places in the API that contacts currently do and you assume responsibility for their lifetime and compliance the same way you would any other contact you create.

## Contact Merging

Identity resolution can never be foolproof. A customer may have multiple devices that they use to contact your organization. Unless either those devices have been assigned to the contact that represents the person or the interaction contains extra information that can be used to associate it to an existing contact, the platform will assume that the new device is a new person and create a new, potentially duplicate contact. When this happens, some manual intervention will be required on the part of the agent.

To clean up these duplicate contacts that represent the same individual, you can merge them together using the [`POST /api/v2/externalcontacts/merge/contacts`](/platform/preview-apis#post-api-v2-externalcontacts-merge-contacts) endpoint. Merging two contacts creates a new “canonical” contact and joins all three into a “merge set.”  The canonical record holds the current state of the contact representing the merge set. The other, non-canonical contacts in the merge set are called ancestor contacts and are made read-only. You can learn more about how to merge contacts and the types of contacts that may be merged in our [developer docs](/commdigital/externalcontacts/contact-merges#merging-contacts).

When an interaction occurs, the platform resolves its identifiers to the current canonical contact mapped to those identifiers and associates the interaction/participant with that contact ID.  Merging two contacts will not update the historical interaction records that were associated with the now-ancestor contacts; those interactions remain associated with the ancestor contacts.

For organizations that integrate with the Genesys Cloud platform, this can present some challenges. If your organization's integrations need to send data to a contact by its ID, that system would need to know the canonical contact ID in order to send the update to the correct record. While this can be done, it would be rather painful to force all users of this feature to have to update their integrations first.

In order to ease the transition for such organizations, the API will transparently redirect some API operations made against ancestor contacts to the canonical contact of its merge set. Any of the following endpoints will exhibit this behavior:

* [`GET /api/v2/externalcontacts/contacts/{contactId}`](/commdigital/externalcontacts/externalcontacts-apis#get-api-v2-externalcontacts-contacts--contactId-)
* [`PUT /api/v2/externalcontacts/contacts/{contactId}`](/commdigital/externalcontacts/externalcontacts-apis#put-api-v2-externalcontacts-contacts--contactId-)
* [`DELETE /api/v2/externalcontacts/contacts/{contactId}`](/commdigital/externalcontacts/externalcontacts-apis#delete-api-v2-externalcontacts-contacts--contactId-)
* [`POST /api/v2/externalcontacts/bulk/contacts`](/commdigital/externalcontacts/externalcontacts-apis#post-api-v2-externalcontacts-bulk-contacts)
* [`POST /api/v2/externalcontacts/bulk/contacts/update`](/commdigital/externalcontacts/externalcontacts-apis#post-api-v2-externalcontacts-bulk-contacts-update)
* [`POST /api/v2/externalcontacts/bulk/contacts/remove`](/commdigital/externalcontacts/externalcontacts-apis#post-api-v2-externalcontacts-bulk-contacts-remove)


Effectively, all contact IDs in the merge set become aliases to the same canonical contact record. The behavior of the [`PUT /api/v2/externalcontacts/conversations/{conversationId}`](/commdigital/externalcontacts/externalcontacts-apis#put-api-v2-externalcontacts-conversations--conversationId-) endpoint will not change.  Specifically, it links the participant to the exact contact ID provided; it will not lookup the canonical contact for the provided contact.

You can find more information about the behavior of ancestor contacts in our [developer docs](/commdigital/externalcontacts/contact-merges#manipulating-ancestor-contacts).  

## Analytics

Previously, analytics queries could filter all records for a specific individual by filtering on a single `externalContactId`.  Now, if you want to filter on a single logical individual (for example, to retrieve their interaction history), you must first obtain one of the `externalContactId`s for the individual, fetch its contact record using [`GET /api/v2/externalcontacts/contacts/{contactId}`](/commdigital/externalcontacts/externalcontacts-apis#get-api-v2-externalcontacts-contacts--contactId-) endpoint, then retrieve the `mergeSet` field from the response.  Using this complete list of merged `externalContactId`s, filter your analytics queries by fetching records that match any of the IDs in the `mergeSet`.  Similarly, if you are grouping analytics records by `externalContactId`, you must map the `externalContactId` to its `mergeSet` as above and group together all records whose `externalContactId`s are in the same `mergeSet`.

Note that this is only required if your organization contains merged contacts. At this time, merging is only done by explicit user actions, so the only way that any merged contacts will be present is if a user or client in your organization performs these actions. Access to the merge action is controlled via the `externalContacts:identity:merge` permission.

## What does all this mean for you?

Well, not much until your organization begins to use the merging features. Merging helps refine your organization's set of contacts over time and makes it easier to see all of the various interactions a customer has had. However, even without using the merge feature, your organization's agents can still get more visibility into your customer's past interactions by using the Customer Journey tab.

Merging is gated behind a separate permission and endpoint, so it's not something that will happen unless your organization explicitly chooses to grant agents permission to do it. Autogenerated contacts do not appear in search results or listing endpoints and should be largely invisible to most existing integrations. Before you start leveraging this functionality, make sure to carefully consider the impact as presented in this post.

## Final thoughts

Single Customer View is a feature we are very excited to deliver to our users. Allowing agents to see all of the touchpoints customers have had with your organization will lead to more empathetic service experiences and improve their interactions with your business. We have worked hard to make it easy to begin using while keeping API compatibility with existing integrations. As your organization begins to use this feature, we would love to hear about your experiences with it!