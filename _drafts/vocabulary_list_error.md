---
layout: post
title: The Taxonomy Vocabulary Performance Issue I Fell Into: loadTree().
---

Taxonomy term management in Drupal 8 has an Achilles heel: `\Drupal\taxonomy\TermStorageInterface::loadTree()`.
Taxonomies that grow too large can WSOD a site with a fatal Out of Memory Error (OOME) when this method is called.

I first ran into it when I attempted to 'List Terms' for a taxonomy with only 12k terms in it although my other vocabularies worked fine. 
I didn't realize at the time it was due to size and I later one post indicating it can happen 
with [as few as 6k terms in the vocabulary](https://www.drupal.org/project/drupal/issues/3101420)! The logs indicated it was an OOME but repeated instances gave different sources for the error; all red herrings.

I began to search the web for answers. One post I found indicated you could [change the number of terms listed at a time](https://www.drupal.org/forum/support/post-installation/2012-11-08/change-the-pagination-of-the-drupal-admin-taxonomy-term#comment-13737248). Surely that would fix the error; right? Unfortunately, no.
`drush config:set taxonomy.settings terms_per_page_admin 25` didn't help although I verified the command *did* change the number of terms displayed per page for other vocabularies. Dropping it down to five terms per page didn't help either! 

Something else was going on so I continued my hunt. I finally stumbled upon the root of the problem, `loadTree()`. This function loads the *entire* vocabulary into memory and, because it is called by the term list admin form (`\Drupal\taxonomy\Form\OverviewTerms::buildForm()`), it can WSOD your page even if try to limit the page to a single term. 😱

There are work-arounds, but this is a pretty ugly hole that Drupal has simply laid a rug over and unwitting site developers and administrators keep falling into.

The first work-around is the classic OOME brute-force fix: increase your PHP's max memory. PHP's max memory limit serves as a not-so-subtle that developers have written inefficient code. If you have to keep increasing this limit you are likely doing something *wrong* although there are exceptions. Most of the time my site's are quite happy living within the 256MB limit and many sites run on much less. Unfortunately, `loadTree()` will quickly consume all that memory and still complain. Giving it more memory does work, to a point. Eventually memory won't be enough; the time it takes to load all that memory will eventually lead to time-out issues. This is simply a losing battle as the vocabulary gets larger.

The rug over the hole: `override_selector` (allows other contrib modules to provide more efficient solutions--i.e. kick the can to contrib)

Build a view that avoids this altogether!


Pages to cite: 

- https://www.drupal.org/project/drupal/issues/2183565
- https://www.drupal.org/project/drupal/issues/763380
- https://www.drupal.org/project/drupal/issues/3158934
