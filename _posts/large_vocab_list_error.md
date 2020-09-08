---
layout: post
title: 'The Taxonomy Vocabulary Performance Issue I Fell Into: loadTree().'
date: 2020-09-07
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

The first work-around is the classic OOME brute-force fix: increase your PHP's max memory. PHP's max memory limit serves as a not-so-subtle that developers have written inefficient code. If you have to keep increasing this limit you are likely doing something *wrong* although there are exceptions. Most of the time my site's are quite happy living within the 256MB limit and many sites run on much less. Unfortunately, `loadTree()` will quickly consume all that memory and still complain. Giving it more memory can work, to a point. [Eventually memory won't be enough](https://www.drupal.org/project/drupal/issues/763380#comment-12613278); the time it takes to load all that memory will eventually lead to time-out issues. This is simply a losing battle as the vocabulary gets larger.

The next work-around is to turn off this feature altogether by setting `taxonomy.settings:override_selector` to `false` and using a view that doesn't use `loadTree()` at all but displays a flat list without hierarchy. This setting is here because [Drupal *knows* they have a scalability problem](https://api.drupal.org/api/drupal/core%21modules%21taxonomy%21src%21TermForm.php/function/TermForm%3A%3Aform/8.9.x):

> \Drupal\taxonomy\TermStorageInterface::loadTree() and
> \Drupal\taxonomy\TermStorageInterface::loadParents() may contain large
> numbers of items so we check for taxonomy.settings:override_selector
> before loading the full vocabulary. Contrib modules can then intercept
> before hook_form_alter to provide scalable alternatives.

Really, Drupal? Kicking performance issues on a core feature like taxonomy to contrib??

In any case, yes, you can turn off this setting and create a term form (with pagination!) that also includes an edit button. I've included one [below](#appendix-manage-taxonomy-terms-view) based on an Islandora 8.x-1.1.0.

Fortunately, there is a better solution that preserves the existing interface! 🎉 Unfortunately it is only available as a patch. ☹️

In the aptly named ticket "Do not use \\Drupal\\taxonomy\\TermStorageInterface::loadTree() in \\Drupal\\taxonomy\\Form\\OverviewTerms::buildForm()", [legolasbo](https://www.drupal.org/u/legolasbo) submitted [a patch in May 2018](https://www.drupal.org/project/drupal/issues/763380#comment-12614808) which was picked up by others but then never merged. There have been a number of re-rolls since and [the July 8th, 2019 patch](https://www.drupal.org/project/drupal/issues/763380#comment-13173830) works perfectly with Drupal 8.9.5.

Applying patches is relatively easy using composer.

Update your `composer.json` to include the following in your 'extra' section:

```
"patches": {
         "drupal/core": {
                 "Fix large taxonomy admin scaling issue": "https://www.drupal.org/files/issues/2019-07-08/763380-61.patch"
         }
 },
```

Then run composer update and clear cache.

I tested this with the Islandora 8.x-1.1.0 release. After the initial provision I updated all my module with composer update and then generated 10k terms in the subject vocabulary. Trying to view that vocabulary resulted in a WSOD (as we would now expect). Applying the patch allowed the subjects vocabulary page to render as it should! 

The trouble with patches is they keep needing re-rolls as core is updated. The sooner we can get this patch in core the better, as far as I'm concerned.

## Appendix: Manage Taxonomy Terms View

Config file: `views.view.manage_taxonomies.yml`:

```
langcode: en
status: true
dependencies:
  config:
    - taxonomy.vocabulary.corporate_body
    - taxonomy.vocabulary.family
    - taxonomy.vocabulary.genre
    - taxonomy.vocabulary.geo_location
    - taxonomy.vocabulary.islandora_access
    - taxonomy.vocabulary.islandora_display
    - taxonomy.vocabulary.islandora_media_use
    - taxonomy.vocabulary.islandora_models
    - taxonomy.vocabulary.language
    - taxonomy.vocabulary.person
    - taxonomy.vocabulary.physical_form
    - taxonomy.vocabulary.resource_types
    - taxonomy.vocabulary.subject
    - taxonomy.vocabulary.tags
    - taxonomy.vocabulary.temporal
  module:
    - taxonomy
    - user
id: manage_taxonomies
label: 'Manage taxonomies'
module: views
description: ''
tag: ''
base_table: taxonomy_term_field_data
base_field: tid
display:
  default:
    display_plugin: default
    id: default
    display_title: Master
    position: 0
    display_options:
      access:
        type: perm
        options:
          perm: 'administer taxonomy'
      cache:
        type: tag
        options: {  }
      query:
        type: views_query
        options:
          disable_sql_rewrite: false
          distinct: false
          replica: false
          query_comment: ''
          query_tags: {  }
      exposed_form:
        type: basic
        options:
          submit_button: Apply
          reset_button: false
          reset_button_label: Reset
          exposed_sorts_label: 'Sort by'
          expose_sort_order: true
          sort_asc_label: Asc
          sort_desc_label: Desc
      pager:
        type: full
        options:
          items_per_page: 10
          offset: 0
          id: 0
          total_pages: null
          tags:
            previous: ‹‹
            next: ››
            first: '« First'
            last: 'Last »'
          expose:
            items_per_page: true
            items_per_page_label: 'Items per page'
            items_per_page_options: '5, 10, 25, 50, 100, 200'
            items_per_page_options_all: false
            items_per_page_options_all_label: '- All -'
            offset: false
            offset_label: Offset
          quantity: 9
      style:
        type: table
        options:
          grouping: {  }
          row_class: ''
          default_row_class: true
          override: true
          sticky: false
          caption: ''
          summary: ''
          description: ''
          columns:
            name: name
            edit_taxonomy_term: edit_taxonomy_term
            status: status
            vid: vid
          info:
            name:
              sortable: true
              default_sort_order: asc
              align: ''
              separator: ''
              empty_column: false
              responsive: ''
            edit_taxonomy_term:
              sortable: true
              default_sort_order: asc
              align: ''
              separator: ''
              empty_column: false
              responsive: ''
            status:
              sortable: true
              default_sort_order: asc
              align: ''
              separator: ''
              empty_column: false
              responsive: ''
            vid:
              sortable: true
              default_sort_order: asc
              align: ''
              separator: ''
              empty_column: false
              responsive: ''
          default: '-1'
          empty_table: false
      row:
        type: fields
      fields:
        name:
          id: name
          table: taxonomy_term_field_data
          field: name
          relationship: none
          group_type: group
          admin_label: ''
          label: Name
          exclude: false
          alter:
            alter_text: false
            text: ''
            make_link: false
            path: ''
            absolute: false
            external: false
            replace_spaces: false
            path_case: none
            trim_whitespace: false
            alt: ''
            rel: ''
            link_class: ''
            prefix: ''
            suffix: ''
            target: ''
            nl2br: false
            max_length: 0
            word_boundary: false
            ellipsis: false
            more_link: false
            more_link_text: ''
            more_link_path: ''
            strip_tags: false
            trim: false
            preserve_tags: ''
            html: false
          element_type: ''
          element_class: ''
          element_label_type: ''
          element_label_class: ''
          element_label_colon: true
          element_wrapper_type: ''
          element_wrapper_class: ''
          element_default_classes: true
          empty: ''
          hide_empty: false
          empty_zero: false
          hide_alter_empty: true
          click_sort_column: value
          type: string
          settings:
            link_to_entity: true
          group_column: value
          group_columns: {  }
          group_rows: true
          delta_limit: 0
          delta_offset: 0
          delta_reversed: false
          delta_first_last: false
          multi_type: separator
          separator: ', '
          field_api_classes: false
          convert_spaces: false
          entity_type: taxonomy_term
          entity_field: name
          plugin_id: term_name
        vid:
          id: vid
          table: taxonomy_term_field_data
          field: vid
          relationship: none
          group_type: group
          admin_label: ''
          label: Vocabulary
          exclude: false
          alter:
            alter_text: false
            text: ''
            make_link: false
            path: ''
            absolute: false
            external: false
            replace_spaces: false
            path_case: none
            trim_whitespace: false
            alt: ''
            rel: ''
            link_class: ''
            prefix: ''
            suffix: ''
            target: ''
            nl2br: false
            max_length: 0
            word_boundary: true
            ellipsis: true
            more_link: false
            more_link_text: ''
            more_link_path: ''
            strip_tags: false
            trim: false
            preserve_tags: ''
            html: false
          element_type: ''
          element_class: ''
          element_label_type: ''
          element_label_class: ''
          element_label_colon: false
          element_wrapper_type: ''
          element_wrapper_class: ''
          element_default_classes: true
          empty: ''
          hide_empty: false
          empty_zero: false
          hide_alter_empty: true
          click_sort_column: target_id
          type: entity_reference_label
          settings:
            link: true
          group_column: target_id
          group_columns: {  }
          group_rows: true
          delta_limit: 0
          delta_offset: 0
          delta_reversed: false
          delta_first_last: false
          multi_type: separator
          separator: ', '
          field_api_classes: false
          entity_type: taxonomy_term
          entity_field: vid
          plugin_id: field
        status:
          id: status
          table: taxonomy_term_field_data
          field: status
          relationship: none
          group_type: group
          admin_label: ''
          label: 'Published?'
          exclude: false
          alter:
            alter_text: false
            text: ''
            make_link: false
            path: ''
            absolute: false
            external: false
            replace_spaces: false
            path_case: none
            trim_whitespace: false
            alt: ''
            rel: ''
            link_class: ''
            prefix: ''
            suffix: ''
            target: ''
            nl2br: false
            max_length: 0
            word_boundary: true
            ellipsis: true
            more_link: false
            more_link_text: ''
            more_link_path: ''
            strip_tags: false
            trim: false
            preserve_tags: ''
            html: false
          element_type: ''
          element_class: ''
          element_label_type: ''
          element_label_class: ''
          element_label_colon: false
          element_wrapper_type: ''
          element_wrapper_class: ''
          element_default_classes: true
          empty: ''
          hide_empty: false
          empty_zero: false
          hide_alter_empty: true
          click_sort_column: value
          type: boolean
          settings:
            format: unicode-yes-no
            format_custom_true: ''
            format_custom_false: ''
          group_column: value
          group_columns: {  }
          group_rows: true
          delta_limit: 0
          delta_offset: 0
          delta_reversed: false
          delta_first_last: false
          multi_type: separator
          separator: ', '
          field_api_classes: false
          entity_type: taxonomy_term
          entity_field: status
          plugin_id: field
        edit_taxonomy_term:
          id: edit_taxonomy_term
          table: taxonomy_term_data
          field: edit_taxonomy_term
          relationship: none
          group_type: group
          admin_label: ''
          label: ''
          exclude: false
          alter:
            alter_text: false
            text: ''
            make_link: false
            path: ''
            absolute: false
            external: false
            replace_spaces: false
            path_case: none
            trim_whitespace: false
            alt: ''
            rel: ''
            link_class: ''
            prefix: ''
            suffix: ''
            target: ''
            nl2br: false
            max_length: 0
            word_boundary: true
            ellipsis: true
            more_link: false
            more_link_text: ''
            more_link_path: ''
            strip_tags: false
            trim: false
            preserve_tags: ''
            html: false
          element_type: ''
          element_class: ''
          element_label_type: ''
          element_label_class: ''
          element_label_colon: false
          element_wrapper_type: ''
          element_wrapper_class: ''
          element_default_classes: true
          empty: ''
          hide_empty: false
          empty_zero: false
          hide_alter_empty: true
          text: Edit
          output_url_as_text: false
          absolute: false
          entity_type: taxonomy_term
          plugin_id: entity_link_edit
      filters:
        name:
          id: name
          table: taxonomy_term_field_data
          field: name
          relationship: none
          group_type: group
          admin_label: ''
          operator: '='
          value: ''
          group: 1
          exposed: true
          expose:
            operator_id: name_op
            label: Name
            description: ''
            use_operator: true
            operator: name_op
            operator_limit_selection: false
            operator_list: {  }
            identifier: name
            required: false
            remember: false
            multiple: false
            remember_roles:
              authenticated: authenticated
              anonymous: '0'
              administrator: '0'
              fedoraadmin: '0'
            placeholder: ''
          is_grouped: false
          group_info:
            label: ''
            description: ''
            identifier: ''
            optional: true
            widget: select
            multiple: false
            remember: false
            default_group: All
            default_group_multiple: {  }
            group_items: {  }
          entity_type: taxonomy_term
          entity_field: name
          plugin_id: string
        vid:
          id: vid
          table: taxonomy_term_field_data
          field: vid
          relationship: none
          group_type: group
          admin_label: ''
          operator: in
          value:
            all: all
            corporate_body: corporate_body
            family: family
            genre: genre
            geo_location: geo_location
            islandora_access: islandora_access
            islandora_display: islandora_display
            islandora_media_use: islandora_media_use
            islandora_models: islandora_models
            language: language
            person: person
            physical_form: physical_form
            resource_types: resource_types
            subject: subject
            tags: tags
            temporal: temporal
          group: 1
          exposed: true
          expose:
            operator_id: vid_op
            label: Vocabulary
            description: ''
            use_operator: false
            operator: vid_op
            operator_limit_selection: false
            operator_list: {  }
            identifier: vid
            required: false
            remember: false
            multiple: false
            remember_roles:
              authenticated: authenticated
              anonymous: '0'
              administrator: '0'
              fedoraadmin: '0'
            reduce: false
          is_grouped: false
          group_info:
            label: ''
            description: ''
            identifier: ''
            optional: true
            widget: select
            multiple: false
            remember: false
            default_group: All
            default_group_multiple: {  }
            group_items: {  }
          entity_type: taxonomy_term
          entity_field: vid
          plugin_id: bundle
        status:
          id: status
          table: taxonomy_term_field_data
          field: status
          relationship: none
          group_type: group
          admin_label: ''
          operator: '='
          value: All
          group: 1
          exposed: true
          expose:
            operator_id: ''
            label: Published
            description: ''
            use_operator: false
            operator: status_op
            operator_limit_selection: false
            operator_list: {  }
            identifier: status
            required: false
            remember: false
            multiple: false
            remember_roles:
              authenticated: authenticated
              anonymous: '0'
              administrator: '0'
              fedoraadmin: '0'
          is_grouped: false
          group_info:
            label: ''
            description: ''
            identifier: ''
            optional: true
            widget: select
            multiple: false
            remember: false
            default_group: All
            default_group_multiple: {  }
            group_items: {  }
          entity_type: taxonomy_term
          entity_field: status
          plugin_id: boolean
      sorts: {  }
      title: 'Manage taxonomies'
      header: {  }
      footer: {  }
      empty: {  }
      relationships: {  }
      arguments: {  }
      display_extenders: {  }
      filter_groups:
        operator: AND
        groups:
          1: AND
    cache_metadata:
      max-age: -1
      contexts:
        - 'languages:language_content'
        - 'languages:language_interface'
        - url
        - url.query_args
        - user.permissions
      tags: {  }
  page_1:
    display_plugin: page
    id: page_1
    display_title: Page
    position: 1
    display_options:
      display_extenders: {  }
      path: manage-taxonomies
    cache_metadata:
      max-age: -1
      contexts:
        - 'languages:language_content'
        - 'languages:language_interface'
        - url
        - url.query_args
        - user.permissions
      tags: {  }
```