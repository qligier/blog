---
title: "Understand the IHE Document Workflow (especially in the Swiss EPR)"
date: 2024-12-07
draft: true
tags: ['Development', 'IPF', 'Camel', 'Java', 'Swiss EPR']
description: "The blog post is describing ."
---

## The two main components

In an IHE community, there are two main components that handle the document workflow: the repository and the registry.
They have a similar responsibility: storing the documents in the community.
The difference is that the repository stores the document content (think the actual file), while the registry stores
the metadata (think the information about the file: who created it, when, etc.).

While they are very similar, IHE had a good reason to separate them: there are use cases where you want only one of 
them, and not the other; in other use cases, you may have multiple repositories than registries, or vice versa.
They can also be deployed on different servers (e.g. a repository in each hospital, and a single registry in a 
central location).

In the Swiss EPR, each community has one registry and one repository that are working together. You contact the 
registry when you want to search for a document, and the repository when you want to download it.

## The different metadata

The registry contains four types of metadata that are useful in the document workflow:

1. DocumentEntry: the metadata of the document itself. It contains information like the title, the author, the 
   creation date and time, the size, the confidentiality level and much more.
2. SubmissionSet: the metadata of a document(s) submission. It contains information like the submission time and the 
   person that triggered the submission.
3. Folder: the metadata of a folder. Folders are used to group documents together, as you would do in a physical 
   folder, or on your computer.
4. Association: the metadata of the relationship between two metadata. For example, the association between a 
   document and a folder (indicating that the document is in the folder), between a document and a submission set 
   (indicating that the document was submitted as part of that submission set), or between two documents (indicating 
   a document references, replaces or appends another document).

All those metadata can be searched in the registry.

In the Swiss EPR, folders are not used; that's one kind of metadata less to worry about.

## The different identifiers

Each metadata has a unique identifier.
entryUUID, uniqueId, logicalId

repositoryUniqueId, homeCommunityId
