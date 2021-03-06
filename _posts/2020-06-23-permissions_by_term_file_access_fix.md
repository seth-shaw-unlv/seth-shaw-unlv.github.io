---
layout: post
title: Permissions By Entity & File Access Fix
date: 2020-06-23
updated: 2020-07-01
---

Some of our digital repository content is not immediately (or ever will be) accessible to the public. The Drupal 8 module [permissions_by_term](https://www.drupal.org/project/permissions_by_term/) and the sub-module permissions_by_entity allows us to create a 'Staff-only' term that, when applied to nodes and media will restrict access to them (and files as an extension of media) to Special Collections staff members.

However, access to files will only be checked if they are in a managed filesystem such as the Drupal private filesystem or one provided by a [Flysystem adapter](https://flysystem.thephpleague.com/v1/docs/) (like the [Fedora Adapter provided by Islandora 8](https://islandora.github.io/documentation/technical-documentation/flysystem/)). When Islandora 8 creates derivate media it places them in the public filesystem by default. This is generally a good idea because it is more performant than other filesystems but then derivatives of restricted items become publicly accessible, which is **not** what we want. A repository manager can change the derivative actions to save them into either the private of Fedora filesystems, but this comes at a performance cost when accessing files that *are* publicly accessible. 

Enter [file_access_fix](https://www.drupal.org/project/file_access_fix). (Credit to [Jonathan Hunt](https://github.com/kayakr) who [introduced me to this module](https://github.com/Islandora/documentation/issues/1134#issuecomment-618093956)!) This handy module will check an entity (node, media, etc.) on update to see if it has any referenced files. If so, it will check if the entity is available by Anonymous users and will move the file to the appropriate filesystem, either public or private, based on it's availability. So, if I create a derivative that is saved in the private filesystem by default, this module will check to see if the media is publicly accessible, if so, it will move it to the more performant public filesystem for me! If I ever decide that the media actually should be restricted, the module will move the file from the public filesystem back to the private one. 

So, all I need to do is setup the private filesystem, change the derivative actions to use the private filesystem, enable these two modules, and setup my 'Staff-only' term and I'm all set.... right? Not quite. We are still missing a few pieces.

First, applying the 'Staff-only' term to a node will not automatically apply it to the related media. Both the node and it's media need the term applied to make it work. Fortunately, some hooks can help with that.

*Note: I like to place local modifications to how Islandora works in a local module called 'islandora_local', a convention I will use in my examples.*

```php
/**
 * Implements hook_node_update().
 *
 * Ensure that all changes to a node's access terms get applied to the related media.
 */
function islandora_local_node_update(NodeInterface $node) {

  // Islandora provides an access terms taxonomy and field that I will use too. 
  if (!$node->hasField('field_access_terms')) {
    return;
  }
  $media = \Drupal::entityManager()->getStorage('media')->loadByProperties(['field_media_of' => $node->id()]);
  foreach ($media as $m) {
    if ($m->hasField('field_access_terms')) {
      $m->field_access_terms = $node->field_access_terms;
      $m->save();
      // This is redundant to the pre-save below, but I wanted to make sure the save took...
      // I'm sure it can be optimized some more.
    }
  }
}

/**
 * Implements hook_media_presave.
 *
 * Inherit the access terms from a related node.
 * This is redundant for the node_update hook, so I should modify this
 * to only trigger on create.... someday.
 */
function islandora_local_media_presave(MediaInterface $entity) {
  if ($entity->hasField('field_media_of') && $entity->hasField('field_access_terms')) {
    // We do a foreach, but there is generally only one.
    // We could probably do a better merge of access terms, but this will do for now.
    foreach ($entity->field_media_of as $media_of) {
      if ($media_of->entity->hasField('field_access_terms')) {
        $entity->field_access_terms = $media_of->field_access_terms;
      }
    }
  }
}
```

With these two hooks in place both my newly created media will get the associated node's access terms applied and, should those terms on a node change, the associated media will be updated too. It isn't perfect, but it will do. (Feel free to let me know how they can be improved.) But we still aren't quite ready yet.

If you try it now all your files in the private filesystem will get moved to the public one, regardless of the applied access terms! 😕 It turns out that if you specifically want to check the Anonymous user's access, it will actually check the logged-in user's access instead! 😱 Fortunately [I discovered a super simple patch](https://www.drupal.org/project/permissions_by_term/issues/3143967)! 

I know, I know... maintaining patches is a sysadmin pain, but that is the cost of this strategy until the patch gets merged. But it gets worse... currently Islandora installs the 1.x version of permissions_by_term, but we want the 2.x version. Until Islandora updates it's dependency, we need to trick composer into letting us install it by updating the required section of our composer.json to include `"drupal/permissions_by_term": "2.24 as 1.x-dev",`. Then do the composer update and apply the patch.

We are almost there... all the pieces are now in place but we need to address hook ordering. The permissions_by_term module updates the grants table during the update hook, but if file_access_fix runs *before* the permissions module hook (the letter "F" coming before the letter "I" and all...) it will use the old permissions. We need to tell Drupal that file_access_fix should happen *later than* the islandora and permissions module update hooks using [`module_set_weight`](https://api.drupal.org/api/drupal/core%21includes%21module.inc/function/module_set_weight/) in an install or update hook:

```php
function islandora_local_install() {
  // Using an arbitrarily large weight will make it run close to last.
  module_set_weight('file_access_fix', 10);
}
 ```
 
Now everything should be in place. Now install your module (or run your update), clear your cache, and enjoy the benefits of making code update media permissions and moving files between filesystems.
 
## Addendum, 2020-07-01
 
This strategy worked great in my testing environments. Unfortunately, it didn't scale well in my production site with more than 200k nodes and half as many media. 🤦‍♂️

After enabling this code on a production site I attempted to update a node's access term and, after waiting 30 seconds, received a 500 error. PHP hit the max execution time. I did a little debugging and noticed that gathering the media didn't cause an issue, but saving the media did. Removing the media presave hook code didn't help either. Simply triggering the media save was enough to overwhelm it.

My first attempt to fix it wrapped the media updates in Batch API operations, but that *still* wasn't enough to fix it. Further testing revealed media saves were taking about three minutes and *file_access_fix* was the cause. Uninstalling the module dropped the *three-minute* media save time down to about *one second*.

But I still want that magical "put the file where it makes the most sense" automation!

I didn't know exactly what in file_access_fix was causing the slow-down, so I decided to take the heart of the module and re-implement it locally:

```php
use Drupal\Core\File\FileSystemInterface;
use Drupal\file\Plugin\Field\FieldType\FileFieldItemList;

/**
 * Make a media's files private/public if necessary.
 */
function update_file_schema(EntityInterface $entity) {
  if (!$entity->hasField('field_media_of') || !$entity->hasField('field_access_terms')) {
    return;
  }

  // Check for access by Anonymous User (uid 0).
  $accessChecker = \Drupal::service('permissions_by_entity.access_checker');
  $accessible = $accessChecker->isAccessAllowed($entity, 0);

  // Load the necessary services.
  $fileSystemService = \Drupal::service('file_system');
  $stream_wrapper_manager = \Drupal::service('stream_wrapper_manager');

  foreach ($entity->getFields(FALSE) as $field) {
    if ($field instanceof FileFieldItemList) {
      foreach ($field->referencedEntities() as $file) {
        // Skip files outside public and private.
        $uriScheme = $stream_wrapper_manager->getScheme($file->getFileUri());
        if (($uriScheme !== 'public') && ($uriScheme !== 'private')) {
          continue;
        }
        $fileIsPublic = $uriScheme === 'public';
        // It should not be both public AND restricted.
        if ($fileIsPublic !== $accessible) {
          // Move it.
          $newUriPath = $stream_wrapper_manager->getTarget($file->getFileUri());
          $newUriScheme = $accessible ? 'public' : 'private';
          $fileTargetUri = "$newUriScheme://$newUriPath";
          $target_dir = dirname($fileTargetUri);
          $fileSystemService->prepareDirectory($target_dir, FileSystemInterface::CREATE_DIRECTORY);
          $movedFile = file_move($file, $fileTargetUri, FileSystemInterface::EXISTS_RENAME);
          if (!$movedFile) {
            \Drupal::logger('islandora_local')->warning("Failed to move file: '@uri'", ['@uri' => $file->getFileUri()]);
          }
          else {
            \Drupal::logger('islandora_local')->notice("Moved file '@uri' to '@new_uri' for Media '@label'", [
              '@uri' => $file->getFileUri(),
              '@new_uri' => $fileTargetUri,
              '@label' => $entity->label(),
            ]);
          }
        }
      }
    }
  }
}

/**
 * Implements hook_media_update().
 */
function islandora_local_media_update(EntityInterface $media) {
  update_file_schema($media);
}

/**
 * Implements hook_media_insert().
 */
function islandora_local_media_insert(EntityInterface $media) {
  update_file_schema($media);
}
```

This method still relies on the permissions_by_term patch I mentioned above, but it works as well as the file_access_fix module did. I *really* wish I knew why file_access_fix was slowing down my media saves. It obviously wasn't the parts I included in this re-implementation, but I can't reproduce it on dev and I would rather not shut-down production to hunt for 🐛s.

*Side note:* Remember the redundancy we experienced earlier with media copying over field_access_terms from the node twice? We can prevent that redundancy by updating the presave hook to check if the media is new or not:

```php
/**
 * Implements hook_media_presave.
 */
function islandora_media_presave(MediaInterface $entity) {
  if ($entity->isNew() && $entity->hasField('field_media_of') && $entity->hasField('field_access_terms')) {
    // We do a foreach, but there is generally only one.
    // We could probably do a better merge of access terms, but this will do for now.
    foreach ($entity->field_media_of as $media_of) {
      if ($media_of->entity->hasField('field_access_terms')) {
        $entity->field_access_terms = $media_of->field_access_terms;
      }
    }
  }
}
```