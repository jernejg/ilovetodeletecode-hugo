+++
date = "2014-05-19T16:10:04Z"
title = "Keep your Mercurial revision history clutter free"
slug = "keep-your-Mercurial-revision-history-clutter-free"
+++
I have been using Mercurial revision control system as a weapon of choice for quite some time now. For controlling changes made to source code, as for backing up to a private repository hosted at Bitbucket I used the basic commands for committing, adding, pushing and pulling files to/from the local /remote repository on a daily basis. And that was sufficient for my needs....until recently. I stumbled upon an interesting blog post titled ["Segregate your commits into tiny topical changes"](http://elegantcode.com/2014/02/15/segregate-your-code-commits-into-tiny-topical-changes/comment-page-1/) that made me think of how my current revision history on a project looks like...

<!--more-->

{{< figure src="/img/Post3_history_example.png">}}

This kind of spaghetti history is very difficult to review. The commits are not topical and there is a lot of cognitive noise just because of backup commits. And I surely wouldn’t want to share code from revision 4 when I send my changes upstream and share them with other developers. It would be nice if you could change history a bit before you push the delivered feature to the public repository, so it would look something like this:

{{< figure src="/img/Post3_better_history_example.png">}}

## How to make changesets mutable in Mercurial

[Mercurial Queues](https://www.mercurial-scm.org/wiki/MqExtension) is an extension that rewrites history. Using MQ we can remove, reorder or fold (merge) committed changesets. This functionality does not come out of the box, first we must configure the extension which is being distributed along with Mercurial. In order to enable the MQ Extension, we modify the local .hgrc file. We can do this directly or through TortoiseHG repository settings dialog:

1. modifying local .hgrc file
{{< highlight ini "style=paraiso-dark" >}}
[extensions]
mq =
{{< /highlight >}}
2. Check TortoiseHG->Repository settings->Extensions->mq 

After the MQ Extension is enabled, Mercurial will include additional commands which we can print with hg help mq, but since this is a TortoiseHG tutorial, I will show you how to work with MQ through TortoiseHG Workbench.

## Goals

Using MQ Extensions we will modify our existing revision history, before we make it public. We will try to:

1. Delete changeset 4 which was backed out.
2. Reorder changesets so that refactoring changeset 5 comes after feature A is implemented.
3. Split a changeset by file or by content using the Shelve tool. This is useful if one commit contains changes related to multiple topics. For example changeset 5 where we did some refactoring also contains logical changes that are part of the feature A implementation.
4. Fold (merge) all changesets that represent topical commit feature A into one changeset.

Since we will modify the revision history directly, we cannot rollback if something goes wrong so it’s better to have a cloned repository for the sake of backup before we start. 

To fully understand Mercurial Queues and the history behind patch management I also suggest reading [Chapter 12](http://hgbook.red-bean.com/read/managing-change-with-mercurial-queues.html) from the official Mercurial guide. But just the introduction without code samples, because they are not up to date.

Long story short, patches deal with handling new code contributions from developers that do not have full commit access to a repository. This is useful on open source projects. But In this case we will use patches to prevent making permanent changes to our revision history and so making it more clutter free.

## Treat changesets as patches

In order to manipulate changesets we must first add them to the Patch Queue. To view the Patch Queue window select "View->Show Patch Queue" from TortoiseHG Workbench:

{{< figure src="/img/Post3_WorkBench.png">}}

If MQ was properly enabled we should see a "Modify History" menu item in the context menu of the selected changeset.

## Goal 1: Delete changeset 4

Select changeset 4 and import it to MQ. After a successful import you should see a new patch was added to the queue named "<revision>.diff". This patch was pushed on top of the repository stack and corresponds to the selected changeset:

{{< figure src="/img/Post3_delete_changeset.png">}}

The next step is to unapply the patch. We can do this from the Patch Queue or from the context menu Changeset 4 "Modify History->Unapply patch":

{{< figure src="/img/Post3_unapply_one_patch.png">}}

When a patch is unapplied its corresponding commit is displayed at the top of the repository graph in a grayed out color:

{{< figure src="/img/Post3_grayed_out_color.png">}}

Now we delete the patch and with this the changeset that corresponds to it:

{{< figure src="/img/Post3_delete_patch.png">}}

## Goal 2: Reorder changesets

Let’s put refactoring changes in a changeset following the Feature A implementation changesets. Select changeset 1 and import it to MQ, then unapply all patches:

{{< figure src="/img/Post3_goal2_reorder.png">}}

Use drag & drop to change the order of changesets. Carry changeset 4 to the end of the queue. You can do this in the Patch Queue window or in the main history view:

{{< figure src="/img/Post3_drag_adn_drop.png">}}

## Goal 3: Split a changeset

Using the shelve tool TortoiseHg "Repository->Shelve" we can move changes between unapplied patches. We can move all the changes made to a file or only chunks of changes inside a file. To move changes we select a target and destination patch in the left and right toolbar then we can use:

1. The list view to select and move changes on a file basis.
2. The text editor to select and move chunks of changes.

{{< figure src="/img/Post3_split.png">}}

## Goal 4: Fold (merge changesets)

Re-apply patch 1 then select patches 2, 3, 5, 6, 7 and click "Fold patches". This will merge selected patches into the topmost applied patch 1:

{{< figure src="/img/Post3_fold.png">}}

A commit dialog will appear, containing comments from all the folded patches separated by a three asterisk syntax. Edit patch message and commit changes:

{{< figure src="/img/Post3_patch_fold.png">}}

Now Re-apply patch 4:

{{< figure src="/img/Post3_reaply_patch4.png">}}

Turn the applied patches back to a regular Mercurial changeset with the "Finish Patch" command:

{{< figure src="/img/Post3_finish_patch_command.png">}}

Our changesets are now ready to be pushed to a public repository and shared with the other collaborators.












