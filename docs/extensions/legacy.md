---
sidebar_position: 99
title: Legacy Extensions
mdx.format: md
---

# Legacy Extensions (Pending Registration)

The properties below predate the extension registry. The
[Oni](https://github.com/crate-works/oni) implementation expects them at the
root level of entity responses. They remain documented here until they are
formally registered as extensions.

## Statistical Counts

- **`counts`** (object): Statistical information about the entity's subtree:
  - **`collections`** (integer): The total number of collections in the subtree
  - **`objects`** (integer): The total number of objects in the subtree
  - **`subCollections`** (integer): The number of nested sub-collections within
  this entity
  - **`files`** (integer): The number of individual files attached to or within
  this entity's structure

## Content Classification

- **`language`** (array of strings): All languages contained in the sub-tree of
entities. Useful for archives that store diverse linguistic data.
  - Example: `["English", "Italian", "Mandarin"]`

- **`communicationMode`** (array of strings): The modes of communication found
in this sub-tree's data, such as speech, song, or sign.
  - Example: `["SpokenLanguage", "Song", "SignLanguage"]`

- **`mediaType`** (array of strings): Media types (MIME types) of files within
the sub-tree.
  - Example: `["audio/wav", "text/plain", "video/mp4"]`

## Access Control

- **`accessControl`** (string): The level of access control required to view or
download this entity.
  - Example: `"Public"`, `"Restricted"`, `"Private"`
