---
title:  "Using GitLab labels"
date:   2016-08-03 21:36:00 -0400
categories: devops GitLab
excerpt: "Use GitLab labels to easily identify merge request state."
---
<br>
![Services](/images/labels.jpg)
<br>
<br>

# Steps

1. Drink the DevOps Koolaid
2. Start managing your infrastructure as code
3. Become afraid of how a small change in the wrong place could devastate your infrastructure
4. Freak out
5. Put controls in place so changes don't get pushed to production without approval
6. Pull your hair out trying to figure out which MRs need your attention

These were the steps we took that resulted in us embracing GitLab labels to direct focus.  Once you start managing your infrastructure as code, using Chef or other tools, you may quickly find
a need to restrict who can merge to master in order to prevent chaos.  

We use [GitLab Enterprise](https://about.gitlab.com/) for source code management.

One of the great features it has is the ability to use labels to help direct focus and workflow.

# GitLab labels
How does GitLab define their label feature?

> Labels provide an easy way to categorize the issues or merge requests based on descriptive titles like bug, documentation or any other text you feel like it. They can have different colors, a description, and are visible throughout the issue tracker or inside each issue individually.

# What labels did we adopt?

Right now we are using 11 GitLab labels in our projects.

![Services](/images/gitlab-labels.jpg)

Each of these has a specific meaning understood by all of our team members.  

| Label                  | Description   | Applied Automatically? |
| -------------          |---------------| -----------------------|
|Blocked	               |Indicates a request should not be merged yet because it is blocked by an external entity.  This could be waiting on another cookbook update or service needs to be updated/deployed. | N |
|Blocked by Events	     |This MR should not be merged because it is waiting for one or more events to complete.  Sometimes potentially impacting changes don't get pushed out if we have an active sporting event.	| N |
|Blocked by Maintenance  |This MR should not be merged until site maintenance is complete. | N |
|Dogfooding	             |This MR should not be merged, but is being internally tested.  This is usually used for our custom Chef Dev Environment. | N |
|Major                   |	This is a major and breaking change as defined by [SemVer](http://semver.org/).  This code should be reviewed very carefully to understand the scope of impact.	| Y |
|Minor                   |	This is a minor change as defined by [SemVer](http://semver.org/).  It adds a new feature to the API.  | Y |
|Patch                   |	This is a fix as defined by [SemVer](http://semver.org/).  | Y |
|Needs Changes           |	Someone has reviewed the MR and the code needs to be updated.  Reviewers comments are sufficient to update.	| N |
|Needs Discussion	       | Someone has reviewed the MR and a discussion is necessary to get this MR back on track. | N |
|Needs Review	           | This is a new MR and no one has looked at it yet.	| Y |
|Ready to Deploy	       | This MR has been reviewed by the correct number of individuals and is ready to be deployed.	| N |

# How do we use labels?

In general, a new MR will get two labels automatically applied to it.  The 'Needs Review' and either Major/Minor or Patch.

The merge request will look a bit like this.

![Services](/images/gitlab-labels-on-mr.jpg)

 Once someone does an initial review, they may add additional labels such as 'Blocked', 'Needs Changes', or 'Needs Discussion'.  After the minimum number of reviewers have looked at the change, the last reviewer will removed the 'Needs Review' label and either merge the MR or add a 'Ready to Deploy' label so that it can be merged at the appropriate time.

# How does this help us?
GitLab displays labels when viewing all open MRs.  This makes it to quickly glance at all MRs that need some form of attention, generally this is those with a 'Needs Review' tag.  Prior to implementing this labeling system I would have to open every MR and read the comments to understand the state.  Now I can direct my attention to only those items that need it. 
