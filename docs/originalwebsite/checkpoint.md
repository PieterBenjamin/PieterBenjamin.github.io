---
layout: page
title: "Checkpoint"
date: 2019-09-25
parent: Original Website
back_to_top: true
---

[Checkpoint](https://github.com/PieterBenjamin/Checkpoint) is a single-file version control tool written in C

As an additional challenge to myself (which I later regretted) I decided to implement this project entirely in C.

My original plan was to get an actual VC system running (even if it were rather simple), and then figure out the differential compression. I borrowed the separate chaining hashtable implementation I had completed in one of my classes (which has requested it not be linked for academic integrity purposes) for simplicity's sake. 

The barebones draft ended up being easier than expected, and consisted of a few simple mappings and bookkeeping information. 

The real fun stuff started when I realized that to keep track of the relationships between checkpoints, I was going to need an n-ary tree... implemented in C. And even better, I was going to need to figure out a way to convert this tree into a one-line representation (to store the tree in a file). Naturally, I ran to the labs and pulled out my favorite keyboard to get this sorted. 

<img src="/assets/images/checkpoint-brainstorm.png">

The above image is the result of my brainstorming. I realized I would need two separate structs, one for the in-memory tree, and one for the on-disk tree. The first was easy enough to comprehend:

```c
typedef struct tree_node {
  struct *tree_node parent;
  char   *cpt_name;
  LinkedList children;
}
```

A node would keep a back-pointer to it's parent, a copy of its name, and a list of its children (since a user may want any number of children to a checkpoint/commit). The more exciting struct is for the on-disk storage:

```c
typedef struct file_tree_node {
  int  name_len;
  char *name;
  int  num_children;
  int  children_offsets[];
}
```

A slight nuance - the second struct was actually implicit. Since you could easily make the disk verison from the physical memory, there was no need to ever actually have that second struct, but the representation is absolutely crucial. It allows for a programmatic way to write/read these arbitrary-width trees. There are two fixed length fields - the name_length, and num_children. 

---

