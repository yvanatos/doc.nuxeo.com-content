---
title: Comments / Annotations
description: See how to handle comments / annotations in Nuxeo
review:
    comment: ''
    date: '2019-12-03'
    status: ok
labels:
    - comments
    - comment-component
    - annotations
    - annotation-component
toc: true
tree_item_index: 1300
---

You can comment any kind of documents, even a comment can be commented (for replies). Annotations work in a same way as annotations are just specialized comments.

Comments (and so annotations) are linked to a specific document (their parent). Annotations are also linked to one of document's blob.

## Core Implementation

### Definitions

Comments in Nuxeo are stored as regular documents of type `Comment`.</br>
Comments hold:

- parent id
- ancestor ids
- author
- text
- creation date
- modification date

[`Comment`](http://community.nuxeo.com/api/nuxeo/latest/javadoc/org/nuxeo/ecm/platform/comment/api/Comment.html) interface has been created to handle comments in Nuxeo. The interface is used as based type in [comment service](#java-api) and also defines how comments are serialized in JSON in order to use them with the [REST API](#json-entity-format).

Annotations are stored as regular document of type `Annotation` which inherit from `Comment`. They hold in addition of comments:

- blob xpath

Like comments, annotations have the [`Annotation`](http://community.nuxeo.com/api/nuxeo/latest/javadoc/org/nuxeo/ecm/platform/comment/api/Annotation.html) interface to use them.

Comments and annotations can have the facet `ExternalEntity`. When they have this facet, it can hold in addition:

- external entity id
- external entity
- external entity origin</br>
With this facet, it is possible to handle external content such as annotations made in [Nuxeo Enhanced Viewer]({{page space='nxdoc' page='nuxeo-enhanced-viewer'}}) for instance.

### Storage

Comments and annotations are stored as a tree structure in a folderish document of type `CommentRoot` which is located right under the commented document. Comments are _special children_ which means they are not copied with the commented document during copy operations but they are snapshotted when the document is checked in.</br>
Please note comments are not restored when a version of the commented document is checked out.

### Permissions

Comments and annotations rely on the [`ACL`](https://community.nuxeo.com/api/nuxeo/latest/javadoc/org/nuxeo/ecm/core/api/security/ACL.html) security model present in Nuxeo. ACLs are automatically inherited by default in `Nuxeo`. So for example, if a user has read access to a document, he also has read access to its children.

By consequence, people who have access to a commented document have read access to its comments and people who create comments and annotations can edit them as they have write access on them.

### Events

All events share the same event context. It contains:
- the parent document in `parentComment` which may be a comment or the commented document
- the comment in `comment_document`
- the comment text in `comment`
- the category `commentCategory` in `category`

A `commentAdded` event is fired when creating a comment or annotation.

A `commentUpdated` event is fired when updating a comment or annotation.

A `commentRemoved` event is fired when deleting a comment or annotation.

### Auto Subscription

When a first comment is added to a document, the document creator is subscribed automatically to the `commentAdded` event on his document(s).

When writing his first comment, a comment author is subscribed automatically to the `commentAdded` event on the commented document.

Once a user unsubscribes from a document, he won't be auto subscribed again to this document.</br>
Auto subscription is turned on by default. It can be turned off by contributing like so:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<component name="org.nuxeo.ecm.platform.comment.service.notification.custom">
  <require>org.nuxeo.ecm.platform.comment.service.notification</require>
  <extension target="org.nuxeo.runtime.ConfigurationService" point="configuration">
    <property name="org.nuxeo.ecm.platform.comment.service.notification.autosubscribe">false<property>
  </extension>
</component>
```

## Usage

### Java API

Nuxeo provides basic implementation of `Comment` and `Annotation` interfaces: [`CommentImpl`](http://community.nuxeo.com/api/nuxeo/latest/javadoc/org/nuxeo/ecm/platform/comment/api/CommentImpl.html) and [`AnnotationImpl`](http://community.nuxeo.com/api/nuxeo/latest/javadoc/org/nuxeo/ecm/platform/comment/api/AnnotationImpl.html).

With help of `Framework#getService`, you can get the [`CommentManager`](http://community.nuxeo.com/api/nuxeo/latest/javadoc/org/nuxeo/ecm/platform/comment/api/CommentManager.html) or [`AnnotationService`](http://community.nuxeo.com/api/nuxeo/latest/javadoc/org/nuxeo/ecm/platform/comment/api/AnnotationService.html) to perform get / create / update / delete operations.

```java
Comment comment = new CommentImpl();
comment.setParentId("PARENT_ID");
comment.setAuthor("Administrator");
comment.setText("A comment!")
comment.setCreationDate(Instant.now());
comment.setModificationDate(Instant.now());

CommentManager service = Framework.getService(CommentManager.class);
comment = service.createComment(coreSession, comment);
comment = service.getComment(coreSession, comment.getId());
comment.setText("New text");
service.updateComment(coreSession, comment.getId(), comment);
service.deleteComment(coreSession, comment.getId());

List<Comment> comments = service.getComments(coreSession, "PARENT_ID");
```

```java
Annotation annotation = new AnnotationImpl();
annotation.setParentId("PARENT_ID");
annotation.setXpath("file:content");
annotation.setAuthor("Administrator");
annotation.setCreationDate(Instant.now());
annotation.setModificationDate(Instant.now());

AnnotationService service = Framework.getService(AnnotationService.class);
annotation = service.createAnnotation(coreSession, annotation);
annotation = service.getAnnotation(coreSession, annotation.getId());
annotation.setText("New text");
service.updateAnnotation(coreSession, annotation.getId(), annotation);
service.deleteAnnotation(coreSession, annotation.getId());

List<Annotation> annotations = service.getAnnotations(coreSession, "PARENT_ID", "file:content");
```

### REST API

Nuxeo provides two web adapters to access comment and annotation services through REST. They are listed on the [web adapter page]({{page space='nxdoc' page='rest-api-web-adapters'}}).

#### JSON Entity Format

Comments are serialized as below:
```json
{
  "entity-type": "comment",
  "id": "00000000-0000-0000-0000-000000000000", // comment id
  "parentId": "00000000-0000-0000-0000-000000000000", // commented document model id
  "ancestorIds": [ // all document ids until higher parent
    "00000000-0000-0000-0000-000000000000",
    "00000000-0000-0000-0000-000000000000"
  ],
  "author": "Joe",
  "text": "A comment example",
  "creationdDate": "1970-01-01T00:00:00Z",
  "modificationDate": "1970-01-01T00:00:00Z",
  "entityId": "...", // the entity id in external system (present when comment is created from another system)
  "origin": "...", // the entity origin (present when comment is created from another system)
  "entity": "...", // the serialized external entity (present when comment is created from another system)
  "numberOfReplies": 0, // present when fetch.comment contains repliesSummary
  "lastReplyDate": "1970-01-01T00:00:00Z" // present when fetch.comment contains repliesSummary
}
```

Annotations are serialized as below:
```json
{
  "entity-type": "annotation",
  "id": "00000000-0000-0000-0000-000000000000", // annotation id
  "parentId": "00000000-0000-0000-0000-000000000000", // annotated document model id
  "ancestorIds": [ // all document ids until higher parent
    "00000000-0000-0000-0000-000000000000",
    "00000000-0000-0000-0000-000000000000"
  ],
  "xpath": "file:content",
  "author": "Joe",
  "text": "A comment example",
  "creationdDate": "1970-01-01T00:00:00Z",
  "modificationDate": "1970-01-01T00:00:00Z",
  "entityId": "...", // the entity id in external system (present when annotation is created from another system)
  "origin": "...", // the entity origin (present when annotation is created from another system)
  "entity": "..." // the serialized external entity (present when annotation is created from another system)
}
```

#### Endpoints

As both comment and annotation endpoints are provided as web adapters, path in tables below are relative to a document (the commented or annotated one), for instance: `/nuxeo/api/v1/id/{docId}`.

Check [Document Resources Endpoint]({{page space='nxdoc' page='document-resources-endpoints'}}) for more information about available document endpoints.

Comments:

| Description                                | HTTP Method | Path                                       | Request Body         | Response                       |
| ------------------------------------------ | ----------- | ------------------------------------------ | -------------------- | ------------------------------ |
| Create a comment on document               | POST        | `/@comment`                                | Comment              | Created comment                |
| Fetch comments of document                 | GET         | `/@comment?pageSize=10&currentPageIndex=0` | /                    | Partial list of comment        |
| Fetch a comment                            | GET         | `/@comment/{commentId}`                    | /                    | Comment with commentId         |
| Fetch a comment by its external entity id   | GET         | `/@comment/external/{entityId}`            | /                    | External comment with entityId |
| Update a comment                           | PUT         | `/@comment/{commentId}`                    | Comment with updates | Updated comment                |
| Update a comment by its external entity id | PUT         | `/@comment/external/{entityId}`            | Comment with updates | Updated external comment       |
| Delete a comment                           | DELETE      | `/@comment/{commentId}`                    | /                    | /                              |
| Delete a comment by its external entity id | DELETE      | `/@comment/external/{entityId}`            | /                    | /                              |

Annotations:

| Description                                    | HTTP Method | Path                                                        | Request Body            | Response                          |
| ---------------------------------------------- | ----------- | ----------------------------------------------------------- | ----------------------- | --------------------------------- |
| Create an annotation on document               | POST        | `/@annotation`                                              | Annotation              | Created annotation                |
| Fetch annotations of document                  | GET         | `/@annotation?xpath=file:content`                           | /                       | Annotation of specified blob      |
| Fetch replies of annotations                   | GET         | `/@annotation/comments?annotationIds=...&annotationIds=...` | /                       | List of replies (deprecated since 11.3) |
| Fetch replies of annotations                   | POST        | `/@annotation/comments`                                     | Array of annotation ids           | List of replies |
| Fetch an annotation                            | GET         | `/@annotation/{annotationId}`                               | /                       | Annotation with annotationId      |
| Fetch an annotation by its external entity id   | GET         | `/@annotation/external/{entityId}`                          | /                       | External annotation with entityId |
| Update an annotation                           | PUT         | `/@annotation/{annotationId}`                               | Annotation with updates | Updated annotation                |
| Update an annotation by its external entity id | PUT         | `/@annotation/external/{entityId}`                          | Annotation with updates | Updated external annotation       |
| Delete an annotation                           | DELETE      | `/@annotation/{annotationId}`                               | /                       | /                                 |
| Delete an annotation by its external entity id | DELETE      | `/@annotation/external/{entityId}`                          | /                       | /                                 |

* * *

<div class="row" data-equalizer data-equalize-on="medium"><div class="column medium-6">{{#> panel heading='Nuxeo Studio Community Cookbook'}}

- [Comment Indexing](https://github.com/nuxeo/nuxeo-studio-community-cookbook/tree/master/modules/nuxeo/comment-indexing)

{{/panel}}</div><div class="column medium-6">
</div></div>
