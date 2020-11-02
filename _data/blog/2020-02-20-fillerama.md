---
template: BlogPost
path: /git-internals
date: 2020-02-20T14:59:36.571Z
title: Git Internals
thumbnail: /assets/header.jpeg
---
Who doesn’t know Git? Most of us are probably using Git or some other source control tool in our everyday life. In this article I will explain how Git works behind the scenes, we’ll dive into the HEAD file, how different Git flow commands work and will explore different Git terminologies. This knowledge will not only improve our Git capabilities but will also introduce deep understanding which will help us build different products in our future career.

In this article I assume you have basic understanding of what Git is, how is it different from other source control tools and that you have used basic git before.

**Disclaimer**: This article is intended for people who want to expand their knowledge in `git-internals` which is a wide topic to understand Git behind the scenes, you won’t learn how to use different git command in this article but rather how they work.

## It all starts from the basics

![](/assets/push-meme.png)

In order to start hacking with Git we will need to use the git init command which will create a local repository for us. This command created a hidden folder with the name of .git which represents the whole repository. From now on, every Git command we will use will be affected by the hidden .git folder that has been just created.

![](/assets/git-01.png)

Let’s dive into the contents of the .git folder, this folder contains different folders & files which responsible for managing your repository:

* Objects directory — A folder that lists all the files history, we will talk briefly about this directory soon.
* Hooks directory — A folder that contains hook scripts that can be run before/after different Git commands.
* Info directory — A folder that contains the exclude file which is responsible for IDE changes, this file is similar to `.gitignore` but it doesn’t intended for sharing.
* Refs directory — A folder that contains all of the configurations of every tag/branch of the repository.
* Head file — a file that serves Git in tracking the current branch, this file contains a pointer to the refs directory.
* config file — the configuration file of the repository, every time we run the Git config command this file is changing.
* description file — contains a description text about the repository.

## Can’t Get You Out of My Head

All of the Git commands which synchronise the local repository and the remote repository are going through a middle station. This middle station has different names such as staging area, cache and index file.

The index file is a binary file which represents kind of a small file system which contains a pointer to different files in the working directory. The responsibility of the index file is to manage the differences between the working directory to the repository itself.

Let’s see now how all of this is actually implemented.

In the beginning of the article when we did git init we saw a new hidden folder named `.git` but this folder didn’t include an index file. In order to create this index file, we will use git add:

![](/assets/git-02.png)

Now we will use git status to see the changes:

![](/assets/git-03.png)

Let’s read the contents of the index file, we will use xxd which is a UNIX command that creates a hex dump of a given file or standard input.

![](/assets/git-04.png)

### HEAD File header

The header stars with the “DIRC” word which represents dir cache Also, the header consist of 12 bytes, the structure of the header is:

* 4 bytes of the Git version
* 4 bytes of the number of records in the file

### HEAD File entry

After the header, we can see that we have a record for every file that we have added. Every record has the following data:

* 8 Bytes of the file’s CTIME (inode/file change time)
* 8 Bytes of the file’s mtime (file modify time)
* 4 Bytes of the file’s device_id
* 4 Bytes of the file’s inode
* 4 Bytes of the file’s permissions
* 4 Bytes of the file’s users uid
* 4 Bytes of the file’s size
* 20 Bytes of the file’s object id
* 2 Bytes of the record’s flag state
* File’s path

In order to view all of those values we can also use the following Git command: `git ls-files - stage - debug`

![](/assets/git-05.png)

## Git == DB

Git is actually a data structure which stores key-value data. For every value we will add in the repository we will get a unique key with whom we could than get the value. Git uses 2 concepts in order to save the data: blobs & trees.

### Blobs

We can use the git bash-object command to get the calculated key for a specified file and also to create a new object in the repository:

![](/assets/git-06.png)

We got SHA-1 of the object we just created which is also named “blob”. Now that we know that Git stores all different data & information in key-value like data structure let’s see where it stores it:

![](/assets/git-07.png)

Interesting… a folder named 84 maybe connected to our SHA-1 hash which starts with the same letters? Let’s get into this folder:

![](/assets/git-08.png)

Git is actually uses the 2 first characters to order the objects in the repository. For every object there is a saved Zlib file.

## Trees

As we said earlier Git uses blobs to save the state of single files. Things are becoming much more complex when we need to save the connections between those files, and a connection between a blob that represents contents to the path. In order to solve this complexity a new concept has been introduced in Git which is named Trees. Tree is another object which saves the contents alongside with the blob’s format, let’s examine the object’s format:

`{file-mode} {object-type} {object-hash}\t{file-name}`

The file-mode field is responsible for saving the permissions of every object in the tree. When Git copies the files to the working directory it needs to save the original permissions thus this information is being saved in the creation of the tree itself.

The following are the possible values for the file-mode field:

* 040000 — for a folder
* 100644 — for read only file
* 100664 — for read/write only file
* 100755 — for an execution file
* 120000 — for a symbolic link file
* 160000 — for a Git link

Every tree in Git can point to other trees, or in different words, a tree serves Git to represent a folder. In this way, Git represents the files & folders structure by a basic tree format.

Let’s verify our understanding on this concept, we’ll create a new folder (in our current directory) and we’ll copy the 1.txt to it:

![](/assets/git-09.png)

Because we didn’t change the contents of the file, the SHA1 calculation of the file has not been changed.

`102633 blob 352806164C31AC7F77CA29CE0F78C13D357F30B0 1.txt`

We can also point our newly created tree to our previous tree.

`102633 blob 352806164C31AC7F77CA29CE0F78C13D357F30B0 1.txt 100644 blob 468A779BA3D7D115B10D71ECD6FB9AC4E50B8E59 2.txt 100644 blob 28AB96C2D9620D1D4F2CC9FD74AFACB1CD16621D newfile.txt 040000 tree 6d2b647a0bb32c9e648ed130afbbfd2608427a23 internal-dir`

## Basics, Basics are important

> I don’t think that the basics that kids need have changed in 10,000 years.

### **Git add**

Git is basically a wrapper on top of the trees concept. Let’s understand this, When we run `git commit`, Git is building a new tree object from the information that is being stored in the index file and saves the object to the objects directory.

Let’s see this in action:

![](/assets/git-10.png)

We can see that a new object of type commit has been created, every objects contains the following data:

* The Tree’s object id
* Information about the commit author
* GPG commit signature
* Commit message

The calculated SHA-1 to the commit object is actually the commit ID we’re already familiar with:

![](/assets/git-11.png)

##### Git log

`git log` helps us in viewing the whole repository history, or in other words the commits objects relations.

Let’s create a new commit:

![](/assets/git-12.png)

After we have created a new commit, an object with the same ID has been created. The tree of that commit points to the same files but on a new blob for the file we just changed. The difference between the old commit object is the parent field, which points to the old commit object. Meaning, by a set of commits we can create history logs:

![](/assets/git-13.png)

### **How git knows what is the working directory?**

In Git we can go back in time, in order to go back in Git we’ll use the `git checkout` command. As you can already understand by using git checkout we replace the commit object and the contents of the index file. So how Git knows where we are now? It uses the .git/HEAD file

![](/assets/git-14.png)

In this example we have seen how `git checkout` replace the contents of the files and the .git/HEAD file which points to the current commit.

Let’s try now to use `git checkout` with branches

![](/assets/git-15.png)

We’ve changed the branch to master, which points to the same commit exactly. Behind the scenes we can see that the value that is written in the HEAD file changed from the value of the commit to the path of a file that matches the master branch. The contents of that path is exactly the same commit we’ve already saw in the previous step.

##### Git branch

In the same way that the master points to the a specific commit, in every branch creation a new file is being created with the contents of a SHA1 encoded file, let’s examine this:

![](/assets/git-16.png)

Because `branch1`is not inside master, they are pointing to the same commit object.

## Summary

In this article we learned how different git commands work behind the scenes. We learned about the index file, it’s purpose, format and how it serves Git. Than, we learned about the building blocks of Git — trees & blobs — their contents & usage.

You can be an expert in Git without having this knowledge, but I’m sure that this knowledge will help you play with Git with more confidence.

![](/assets/footer.png)
