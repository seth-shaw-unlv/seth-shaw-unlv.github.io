---
layout: post
title: Content Access Control Solutions Investigation
date: 2021-02-19
updated: 2021-02-24
---

My [recent investigation into abysmal digital asset management site performance]({% link _posts/2021-02-18-performance_analysis.md %}) under load identified [Permissions by Term module (permissions_by_term)](https://www.drupal.org/project/permissions_by_term) as the primary culprit. Some preliminary exploration found at least one optimization that _improved_ performance, but not sufficiently. [Others have also noted performance issues](https://www.drupal.org/project/permissions_by_term/issues/3126542) and made some suggestions, although the suggestions appear abandoned. I could spend time developing more optimizations but, given that [a patch I submitted in June 2020](https://www.drupal.org/project/permissions_by_term/issues/3143967) was never reviewed, I have little hope that my patches would be merged.

This report documents my investigations into alternate solutions.


# Setup

I used the same virtual machine setup for the performance analysis (see the link above) with the addition of a new snapshot that includes a few media to test each solutions' support for controlling media access.

# Method

For each module, I performed an initial review of existing documentation, open issues, and code. I then (attempted) to install and configure each module to restrict certain nodes and their associated media to manually test node and media exclusion from direct links, views, and searches.[^1] I initially planned to move forward with the same TTFB tests used for the performance analysis, but none made it quite far enough to justify it. 😬

# Module Reviews

*Note: The modules below are listed alphabetically and not necessarily in the order in which they were tested. Key takeaways are **emphasized**.*

## [Access by entity](https://www.drupal.org/project/access_by_entity)

This module allows sites to restrict access to particular entities. Unlike other modules, you select roles that _should not_ have access, rather than those that _should_. 

Unfortunately, [the module currently breaks views](https://www.drupal.org/project/access_by_entity/issues/2917318), including the content management and dblog pages. 🤦‍♂️ There is a patch, but it has been a work in progress over the last few years. It only received a workable patch in December 2020 which has yet to be merged. Even then, views that would otherwise include a restricted item fail with an invalid SQL query.

Despite the current (three-year-old) version being broken, the module could be fixable and has recently received some community attention, but the module's maintainer has been MIA on Drupal.org for nearly a year and hasn't touched this module since shortly after the module's last release in 2017. **Someone would have to claim the maintainer role and shepherd an unknown number of fixes to make it stable again.**


## [Advanced Access](https://www.drupal.org/project/adva) / [Role Access Control](https://www.drupal.org/project/rac)

Advanced Access (adva) aims to extend the existing node access controls to other entities. The Role Access Control module (rac) extends Advanced Access to provide role-based access controls.[^2] 

Adva is still on release candidate 9 and, when I began this report, the most recent commit was 6 months old (July 28th, 2020) with a 5 month old [an RTBC critical bug patch](https://www.drupal.org/project/adva/issues/3143268) waiting for review. I reached out to the developer who admitted their neglect of adva and rac due, understandably, to a job change. They intend to continue maintaining them and merged the critical patch at my prompting. However, future responsiveness is still to be determined.

My initial attempts to get it working were unsuccessful. My test VM started throwing a RouteNotFound exception when attempting to edit a node to add the Role reference. Further testing discovered this error wasn't due to adva or rac, but rather a bug in Islandora 1.1.0. Fortunately, I can easily use the patch provided by [Islandora PR 781](https://github.com/Islandora/islandora/pull/781) via composer or upgrade to the dev branch.[^composer-patches] **The primary concern is that [media with role references are not being restricted as I would expect](https://www.drupal.org/project/adva/issues/3197839).**

## [Content Access](https://www.drupal.org/project/content_access)

The module is used on numerous sites, which is promising, despite only having an alpha release. It appears that the primary blocker for [a 1.0 release](https://www.drupal.org/project/content_access/issues/3143952) is issue [2897104](https://www.drupal.org/project/content_access/issues/2897104) that impacts sites with multiple translations of a restricted node. This is not something our site will soon encounter, but it may be important to other sites.

It appears to only control access to nodes, but it does support setting role-based access for individual nodes. Unfortunately, changing a single item's permissions requires a full permissions rebuild which is an expensive operation! 😱 **This will not scale for our repository, at all.**

## [Embargoes](https://github.com/discoverygarden/embargoes)

This module was created by [Bryan Brown <!-- aka Paxton 😁 --> at Florida State University Libraries](https://github.com/fsulib/embargoes)  specifically for embargoing scholarly works in Islandora for an institutional repository. Since then Discovery Garden forked the module and began updating it.[^3]

I spoke with Bryan Brown and Daniel Aitken on the Islandora Slack about the future of the module. Bryan recommended using the Discovery Garden fork. There is a consensus that embargos should be migrated from config entities to content entities and embargo types (currently date and IP range) should be shifted to plugins instead of services. Although this would take significant development time to fit our needs (including creating a role-based embargo type), this solution is geared towards supporting Islandora idioms and is likely to be picked up by the community.

Initial tests showed that applying the embargoes worked well for restricting either a node and its associated media/file or just the node's media/file. Unfortunately, [the module only implements hook_ENTITY_TYPE_access for nodes, media, and files](https://github.com/discoverygarden/embargoes/issues/16#issuecomment-774340724) which is ignored during entity queries and views which means they will still show up in searches and other item listings. **The module either needs to implement [hook_query_TAG_alter()](https://api.drupal.org/api/drupal/core%21lib%21Drupal%21Core%21Database%21database.api.php/function/hook_query_TAG_alter/8.9.x) or start employing node grants.**

Another question is how to represent the restrictions in Fedora. Embargoes creates a new entity (currently config entity, but eventually a content entity) which would need to either be transformed to a string or XML value with an appropriate RDF predicate using a JSON-LD alter OR sent to Fedora as separate entities with the objects storing references instead of the string value. Alternatively, we simply ignore it and use a separate field value in parallel to the embargo stored as a field on the object.

## [Group](https://www.drupal.org/project/group)

This module "allows you to create arbitrary collections of your content and users on your site and grant access control permissions on those collections." That stated, it isn't very clear how to _use_ it without finding some tutorials.

Attempting to use it makes it clear that the quote above is literal. It is a collection of content and users. I could not find a simple way to move content from one group to another. You can't take existing items and simply 'tag' them as belonging to a group to hide them and then, later, remove that relationship. *Content is created within a group, for that group.* **There might be some magical configuration combination to make this work for restricting certain content that can eventually be released, but I haven't found it.**

## [Node View Permissions](https://www.drupal.org/project/node_view_permissions)

This module simply allows sites to state if a user can view their own or any of a particular content type. _That's it._ It is only included in the list as it frequently shows up when searching this topic but **it can be disregarded**.

## [Taxonomy Access Control Lite](https://www.drupal.org/project/tac_lite)

This module has a somewhat awkward setup but does the job of restricting access to nodes and keeping them out of views as expected. However, attempts to restrict the associated media fall flat. There is [an issue ticket for media support](https://www.drupal.org/project/tac_lite/issues/3007291), but it was postponed for a core issue that appears abandoned. 

Even more concerning is another ticket entitled "[Question for TAC Lite users: Should I use TAC Lite on a new project?](https://www.drupal.org/project/tac_lite/issues/3047966)" posted in April 2019 with _no responses whatsoever_. The issue queue has a response rate of 0% for nearly the last two years according to the Drupal statistics graph for the module. The only commits to the repository since 2017 were those necessary for Drupal 9 readiness; no bug-fixes or new features. All the issue thread activity appears to be potential adopters asking questions that fail to be answered. **Aside from the technical issues, I think we can consider this module's community dead.**

# Summary

None of the modules I've reviewed are a scalable ready-to-use replacement for permissions_by_term. *Any one of them* would require some degree of enhancement in addition to writing an update hook before they could be adopted. However, two stand out as possible solutions: [Advanced Access](#advanced-access--role-access-control) (in conjunction with its companion module Roles Access Control) and [Embargoes](#embargoes). Both need additional testing at scale once critical issues are resolved.

## Advanced Access + Roles Access Control

The Advanced Access + Roles Access Control module, functionally speaking, is the closest to our existing setup. The existing taxonomy reference field we use on nodes and media for referencing the Staff-Only term would be replaced with a role reference field referencing our existing Special Collections Staff role. This would make the migration relatively simple: 
1. create the new field,
1. add the necessary module configuration,
1. find every reference to the Staff-Only term and create a corresponding role reference value for the same entity,
1. drop the Islandora Access reference field, then
1. uninstall permissions_by_term.

This module also aligns closely with the direction Drupal Core has taken with access controls which means we may be able to phase it out if Core extends node grants to other entities.

The complications with this solution are the uncertainty of maintainer and community support for the module and the Media accessibility bug. The maintainer has indicated a desire to continue maintaining it despite a recent hiatus, but only time will tell. I have identified [the cause of the media display issue](https://www.drupal.org/project/adva/issues/3197839#comment-14002722), but a solution needs time to develop and test.

## Embargoes

The Embargoes module was built by an Islandora community member for the Institutional Repository segment of the Islandora community and has recently been taken up by a consulting group that is very invested in the community. It already supports indefinite, release-date, and IP-based restrictions. It is likely to continue being developed and supported as more and more of the community migrates to Islandora 8 for institutional repositories. 

The complications with this module are that it requires significant refactoring and works quite a bit differently than our current solution. Unlike the most other solutions, it requires separate entities to store the embargo information. Our existing reference field to the Staff-Only taxonomy term would need to be replaced with links to new embargo entities. The refactoring and enhancements include changing the embargo entities from configuration entities (which would swamp Drupal's configuration system if it is used at scale) to content entities. Even then, we still need to consider how to represent these embargoes in Fedora for long-term preservation. Further, while the access restrictions _mostly_ work, there are still some bugs that reveal restricted items when found via Views (e.g. search and related items lists) that need to be fixed.

## Which One?

To be short, both of the prime candidates have their issues and require further development time and testing before receiving a stamp of approval. The question is _which_: adva+rac, having a more questionable community but seemingly smaller technical lift, or embargoes, with a stronger likelihood of community support but a heftier technical lift that is further outside The Drupal Way?

Ideally, we would do both to better compare their performance at scale, but this issue is blocking progress on the other pre-launch tasks, so we can only afford to invest in one right now. I am inclined to attempt fixing Advanced Access which, if it works, would move the launch forward more quickly. Adopting embargoes won't be any more difficult to implement if we choose to pursue it later.[^embargoes-timeline]

_Update (2021-02-23): During the [Islandora Open Meeting on February 23rd, 2021](https://docs.google.com/document/d/1nZc9oMjFa1aklM9FdizvM-Trtp4jFp1WCPSJR-oL-RE/) we learned that ASU has been using Group for [collection membership](https://gist.github.com/elizoller/9d0135c4c122cbec20c72e68a95ac8d8). This isn't the same use-case we are looking at but it appears more reasonable than it initially seemed. They also admit that the module's documentation is lacking and it took a lot of code-reading to understand. So, there may actually be some magical setup where it would work for us, but it would take significant effort. If Advanced Access doesn't pan out we can investigate Group along-side Embargo._

_Update (2020-02-24): [The slides from my presentation on this and performance testing at the Open Meeting are available as PDF](/files/2021-02-23_Islandora_Open_Meeting_Performance_Testing_and_Content_Access_Control.pdf) and [a recording of the presentation](https://youtu.be/tKQIdYjsVDo) is available on the Islandora Foundation YouTube site._

# Additional Resources

- [Node Access Rights (Drupal 8.9 API)](https://api.drupal.org/api/drupal/core%21modules%21node%21node.module/group/node_access/8.9.x)
- [function node_node_access (Drupal 8.9 API)](https://api.drupal.org/api/drupal/core%21modules%21node%21node.module/function/node_node_access/8.9.x)
- [Content locking (anti-concurrent editing)](https://www.drupal.org/project/content_lock)
- [Media Private Access](https://www.drupal.org/project/media_private_access)
- [Change record: "Added an entity query access API"](https://www.drupal.org/node/3002038)
- "[HOW DO YOU RESTRICT ACCESS TO CONTENT IN DRUPAL 8? 6 MODULES THAT WILL DO THE JOB FOR YOU](https://www.optasy.com/blog/how-do-you-restrict-access-content-drupal-8-6-modules-will-do-job-you)"

## Notes

[^1]:
     Initial research into possible modules indicates that most only apply their access controls to content types or individual nodes, **not** to associated media. For example, the permissions_by_term module doesn't support media itself but, unlike others, it does provide a sub-module to extend support to media entities.

[^2]:
     They also note on the project page an intent to integrate with Taxonomy Access Control Lite, which would be closer to our existing permissions_by_term solution (but scalable)! However, TAC appears to be lacking support, so how realistic that is is unclear.

[^composer-patches]:
     See [composer-patches](https://github.com/cweagans/composer-patches) for how to apply patches with composer. GitHub allows you to get a patch for a pull request by adding '.patch' to a pull request URL. E.g. 'https://github.com/Islandora/islandora/pull/781.patch'.

[^3]:
     Notes about the improvements to date are available on [a PR for merging the DG changes](https://github.com/fsulib/embargoes/pull/15) back into the FSU repo.

[^embargoes-timeline]:
     If we do pivot to embargoes the development will probably occur *after* the site's initial public launch. This would require us to *temporarily* remove existing restricted content from the system and run it *without* a content access control solution for the time being. Restricted content would only be placed back in the system *after* we have thoroughly tested the revised embargo solution.