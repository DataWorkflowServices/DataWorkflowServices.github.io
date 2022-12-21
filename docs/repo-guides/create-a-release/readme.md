---
authors: Dean Roehrich <dean.roehrich@@hpe.com>
categories: release
---

# Create a Release: Notes about creating a release

The following notes describe how to use the GitHub release process to create a release of a repo, and the workflows in place to guide this process.  This includes the use of release branches and annotated release tags.

## GitHub Release Process

See [About releases](https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases) for the GitHub documentation on releases.

A release can be created on GitHub by clicking the `Create a new release` button in the repo.  Select an annotated tag for your release. **Do not use this interface to create a tag** because that tag will not be annotated and it will fail the repo's workflow steps.  See the steps below to create an annotated tag, then proceed to the remaining steps in the GitHub documentation.

## Release branches

Release branches are named `releases/vX`, where X is 1, 2, etc.  In the usual case, all version 1.x.y release tags will go into the `releases/v1` branch, and all version 2.x.y tags will go into the `releases/v2` branch.  Fine-grained branches such as `releases/v2.1` for all version 2.1.y tags may be used if necessary.

The repo has a "branch protection rule" on the branch pattern of `releases/v*` to enforce the use of pull requests and reviews on release branches.  See [Managing a branch protection rule](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/defining-the-mergeability-of-pull-requests/managing-a-branch-protection-rule).

## Release tags

Release tags are named in patterns like `vX.Y.Z` or `vX.Y.Z-alpha` and are applied to the release branches.

The repo has a "tag protection rule" on the tag pattern of `v*`.  Only users who have admin or maintain permission may create or delete protected tags.  See [Configuring tag protection rules](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/managing-repository-settings/configuring-tag-protection-rules).

**IMPORTANT** The tag must be applied to a commit that is entirely within the release branch, so that it is not visible from the main branch.  If it is visible on the main branch, then image tags from the main branch will collide with image tags on the release branch.  This means that after you create the release branch you may have to add another commit to it simply to have a place to attach the tag.

<details>
<summary>Verify tag visibility</summary>
    After you've created the tag in your workarea, but before you've pushed it to the repo, verify that it is visible only on the release branch and not on the main branch.

```console title="Look for tags on the main branch"
git checkout main
git describe --match="v*" HEAD
```

```console title="Look for tags on the release branch"
git checkout releases/v1
git describe --match="v*" HEAD
```
</details>

### Annotated tags

Our process requires that release tags be annotated.  An annotated tag contains information about who created it and when it was created.  Annotated tags may also be signed.

The repo contains a workflow which verifies that any pushed tag is an annotated tag.

### Creating an annotated tag.

You can use GitHub Desktop to create an annotated tag, or you can create the tag with the `git tag -a` command.

```console title="Example annotated tag"
git checkout releases/v1
git tag -a v1.0.2 -m "My release 1.0.2"
git push origin --tags
```

<details>
<summary>Verify that a tag is annotated</summary>
When a tag is annotated, its details will be displayed before the commit the tag references.

```console title="Show a tag"
git show v0.0.6
```

If the tag is annotated then its details will be shown before the commit is shown.


```conf title="An annotated tag"
tag v0.0.6
Tagger: Dean Roehrich <dean.roehrich@hpe.com>
Date:   Wed Dec 14 15:15:46 2022 -0600

Experimental release #8

commit 91fea7d02a5e7196ca8b4686deb33898109fd8c0 (HEAD, tag: v0.0.6)
Author: Dean Roehrich <dean.roehrich@hpe.com>
Date:   Wed Dec 14 14:44:15 2022 -0600

    Rename the workflow that verifies tags. (#82)
    
    Signed-off-by: Dean Roehrich <dean.roehrich@hpe.com>

[...]
```
</details>



