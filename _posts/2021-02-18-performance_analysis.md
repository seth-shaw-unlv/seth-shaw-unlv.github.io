---
layout: post
title: Islandora Performance Analysis
date: 2021-02-18
---

Currently, the our new Islandora-based digital asset management system (yet to be released) is returning pages from a cold cache (meaning Drupal hasn't pre-computed parts of the page) at an unacceptably slow rate. Some slowness is acceptable from a cold cache as most pages will be cached and return quickly (within a fraction of a second). However, our site contains a long tail of content and we can't anticipate what users will search for. These pages do not have the benefit of page caching and so need a passable loading speed without it.

While we have not determined specific performance goals for the site yet, these statements from "High Performance Drupal" seem appropriate with a minor edit for the types of pages of interest: “The maximum page load time across the entire site should always remain below eight seconds. [Digital Object and search] pages should have a maximum page load time of five seconds. The front page of the site should have a maximum page load time of three seconds.”[^high-performance-drupal]

Just how bad is it? Some search pages are taking up to 1.2 minutes to load! 😱

# Setup

I've built a local development VM based on the Islandora 1.1.0 image using the [Islandora Playbook](https://github.com/Islandora-Devops/islandora-playbook) with the [ArchivesSpace Drupal module](https://www.drupal.org/project/archivesspace) (1.0) installed along with our local theme (drupal8_parallax_theme aka special) and modules (islandora_local and contentdm_migrations). My available disk space isn't sufficient for large scale loading of repository files, so initial testing focused metadata-only loads of digital objects and archival descriptions.

Additionally, I disabled the JS/CSS aggregation and turned off cron so they wouldn't inadvertently impact the tests.


# Method

The primary measurement for the initial tests is a ‘time to first byte’ (TTFB) measurement of pre-selected nodes and search queries. TTFB was measured by executing a curl command, preceded by the ``drush clear-cache`` command before each request to ensure a cold cache.[^1] I took 5 measurements of each URL in each scenario so we can see the minimum, maximum, and average response times. The nodes were semi-randomly selected, adjusted to preference a mix of single objects and parent objects with multiple children. I setup small shell scripts to iterate through the test sets so I didn't have to manually reset between tests.[^test-sets]

For example, the following script was run to test search-page loading across four different themes:

```sh
#!/bin/bash

# Each theme to test
themes=( 
  stark 
  bartik
  carapace
  drupal8_parallax_theme
)

# Search pages to test
urls=(
  "http://localhost:8000/search?keys=Jerry+Jackson"
  "http://localhost:8000/search?keys=%22Las%20Vegas%22&f%5B0%5D=type%3AGeographic%20Location"
  "http://localhost:8000/search?keys=%22Hughes,%20Howard%22"
  "http://localhost:8000/search?keys=%22Spruce+Goose%22"
  "http://localhost:8000/search?keys=XF-11+Crash"
  "http://localhost:8000/search?keys=XF-11%20Crash&f%5B0%5D=subjects%3AHughes%20XF-11"
)
for theme in "${themes[@]}"
do
  # Set the theme
  drush cset -y --quiet system.theme default $theme
  
  for URL in "${urls[@]}"
  do
    # Run each search 5 times.
    for run in {1..5}
    do
      # Reset the cache
      drush cr --quiet;
      # Perform the search, discard the page, and print the result.
      curl -o /dev/null -sS -w "$theme\t$URL\t$run\t%{time_starttransfer}\n" $URL;
    done
  done
done
```

I am conducting four levels of load for the tests: digital collections taxonomies, the "Welcome Home, Howard" digital collection, all of the digital collections, and all of the digital collections plus the archival descriptions. This allows us to see the impact of object count on loading times, if any. Each load was followed by a SOLR indexing run to ensure everything is available for the search queries.[^2] I then created a virtual machine snapshot for each load level so I could restore them to run (or rerun) tests for the various levels as needed.

# Tests

## Theme Performance

Our site includes quite a bit of additional coding, especially for displaying search results, which could potentially impact site performance. To check if this was the case my first tests compare themes: stark (minimum HTML and CSS), bartik (Drupal's default theme), carapace (the theme provided with Islandora 1.1.0), and special (our local theme).

First I first ensured the enabled blocks were consistent across the themes I'm testing. Blocks can be isolated for individual testing later if needed.

Testing results revealed that, while the theme does have an impact on the TTFB, the degree of difference is not nearly as significant, 5.3% at full load, as other factors appear to be.[^3] Figures 1 and 2 illustrate that the themes, while slowly diverging as load increases, still trend together.

<figure class="chart">
  <img src="/images/2021/performance_fig_1.png" alt="1－Average TTFB for a digital object page at increasing load levels.">
  <figcaption>Fig. 1－Average TTFB for a digital object page at increasing load levels. Each color is a separate theme.</figcaption>
</figure>

<figure class="chart">
  <img src="/images/2021/performance_fig_2.png" alt="Average TTFB for a search page at increasing load levels.">
  <figcaption>Fig. 2－Average TTFB for a search page at increasing load levels. Each color is a separate theme.</figcaption>
</figure>

It appears evident that some other factor is responsible for the drastically increasing load times.

## Permissions Performance

To better understand these large load times I used the XHProf profiling tool to see which portions of the code have the greatest durations. Although various pieces of code are reused by other portions of the system, it was clear that the permissions system was consuming most of the processing time with several expensive database calls. Comparing times with Permissions by Term (and the associated sub-module, Permissions by Entity) enabled and without is quite telling:

<figure class="chart">
  <img src="/images/2021/performance_fig_3.png" alt="Shows the average time to first byte for the different types of pages (search, simple objects, and complex objects), using the Special theme, with and without Permissions by Term (PBT) enabled.">
  <figcaption>Fig. 3－Shows the average time to first byte for the different types of pages (search, simple objects, and complex objects), using the Special theme, with and without Permissions by Term (PBT) enabled.  Note that the line for simple objects without permissions is so in line with complex objects, that it is completely obscured.</figcaption>
</figure>

As illustrated in figure 3, the time to first byte increases while using Permissions by Term corresponding to the increase in load, whereas the trend line for the pages without a permissions module remains relatively flat.

Exploratory work in modifying the permissions_by_term module showed that at least one code fix could reduce the module's TTFB by ~10 seconds on DAMS (from 27 seconds to 17). Unfortunately, that optimization won't be enough on its own.

This result alone is enough for us to shift from performance testing to an investigation of content permissions solutions. We will either need to find an alternative permissions module (the ideal solution), attempt to make permissions_by_term acceptably performant (and pray the module maintainers accept my patches), or developing our own lighter-weight alternative (the least attractive option).[^4]

# Summary

There are few things on the internet as a slowly loading page. This process illustrates how our first assumptions are not always accurate. I could have spent a lot of time chasing minor gains by tweaking my theme without addressing the real issue. Setting up a testing environment and putting various aspects through their paces is worth the work. Addressing the performance issues with our content permissions is my next step, but there are plenty of other things we can try to further improve performance. I'll be sure to put them through their paces.

# Additional Notes

## SOLR Performance

In late January I reviewed the SOLR logs and found that most queries returned in less than 100 milliseconds, a drop in the bucket compared to the 30 second to 1.2 minute search page TTFB. The few uncommon outliers were closer to 500 milliseconds although one query did hit 1 second. Again, only a small fraction of these long page result times. While there might be [some optimizations for SOLR we can try](https://lucene.apache.org/solr/guide/8_0/taking-solr-to-production.html#run-the-solr-installation-script) in the future, the potential improvements in page loading time is marginal compared to the other existing issues.

## Slow Queries

Early in January a system-admin enabled the slow-query log on our future production site. Nothing was appearing in the log by late January. Given that, I suspected the problem wasn't single large queries, but too many small individual queries per request (at least for the search) that simply add up. This turned out to be exactly the case. Permissions by Term was repeatedly making queries that missed the slow-query threshold but, altogether, became painfully slow.


## Potential Areas for Performance Enhancement/Review

There are a number of webpages that provide suggestions for improving Drupal performance. Often they repeat each other. Below is a list of their suggestions, with additional commentary, in the order in which they were first seen. We would have tested a number of these had we not identified our culprit as quickly as we did. We may yet investigate some of these in the future.

-   Ensure Page & Block caching are enabled (moot for the cold-cache case we are testing)
-   Enable CSS and JS aggregation (applies for over-all page loading time; impact on TTFB unclear but possibly slows it down by a few microseconds)
-   Enable Twig caching
-   [Views Litepager](https://www.drupal.org/project/views_litepager). This module increases view speed by only providing previous and next links. (May or may not apply to views infinite scroll; impact should be tested)
-   [Boost](https://www.drupal.org/project/boost). This module generates static HTML pages for anonymous user. This could drastically improve page loading for anonymous users, although it would need testing to determine disk space and processing time requirements. It also likely won't apply to search pages (the biggest performance issue we have right now).
-   Review site for modules to disable, e.g.:
    -   Statistics
    -   PHP Filter
-   Using IIIF/Cantaloupe for images instead of stored items
-   Use [Syslog](https://www.drupal.org/docs/8/core/modules/syslog/overview) for the error log
-   [Search without loading entities](https://www.drupal.org/docs/8/modules/search-api-solr/search-api-solr-howtos/create-a-search-view-that-doesnt-load)
-   Offloading some hook code to the [Queue API](https://api.drupal.org/api/drupal/core!core.api.php/group/queue/8.9.x) for background processing (e.g. moving media from private to public filesystem).
-   Using a MySQL slave for Views or programmatic entity queries.
-   Avoiding node_access in Views (can introduce security issues).
-   [Memcache](https://www.drupal.org/project/memcache) (isn't likely to impact local-dev TTFB testing; it's benefit comes from preserving the MySQL database for new requests during live-site traffic OR if the Entity Cache has been warmed).
-   Replacing MySQL with MongoDB for the Entity/Field storage, useful for highly dynamic sites with lots of content entities, like ours. (It would be a _drastic_ change that may limit available contrib modules; requires an extensive review and perhaps a 'last resort' option.) _Not a full review of the site, but permissions_by_term, which we currently use, would not support this use-case._

## Notes

[^high-performance-drupal]:
     Sheltren, Jeff, et al. *[High Performance Drupal: Fast and Scalable Designs](https://www.oreilly.com/library/view/high-performance-drupal/9781449358013/).* 1st ed., Sebastopol, CA, O’Reilly Media, 2013.

[^1]:
     ``curl -o /dev/null -sS -w "$theme\t$URL\t$run\t%{time_starttransfer}\n" $URL``

[^test-sets]:
     For the curious, I'm posting [the test scripts and the collected test data in a zip file](/files/2021-01_performance_analysis_tests.zip).

[^2]:
     The indexer was stopped early for the full archival descriptions load. The current performance issues cause the indexer to run ~6 times slower than it usually would and would have taken a full week to complete. Reviewing the SOLR logs indicated that, at the DAMS current content load, SOLR was returning most results in under 100 milliseconds which barely impacts the overall TTFB. Given the relatively long time required for a full index and it's relative un-importance to overall TTFB, I decided to expedite the testing process by stopping the indexer at ~61%, which was sufficient to contain all taxonomy terms, digital collections objects, archival description, and a significant portion or archival component objects. See the SOLR Performance section in "Additional Notes."

[^3]:
      The standard deviation between the themes for loading a digital object is only 0.96 seconds at the greatest load, compared to the average load time of 17.95 seconds.

[^4]:
     See "[HOW DO YOU RESTRICT ACCESS TO CONTENT IN DRUPAL 8? 6 MODULES THAT WILL DO THE JOB FOR YOU](https://www.optasy.com/blog/how-do-you-restrict-access-content-drupal-8-6-modules-will-do-job-you)" for additional permissions modules we can test. From this list, [Taxonomy Access Control Lite](https://www.drupal.org/project/tac_lite) looks promising at first glance and my initial performance tests show no performance impact. However, it does not appear to be actively maintained and it still needs to be reviewed for Media support.
