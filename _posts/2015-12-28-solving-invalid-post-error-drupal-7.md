---
layout: post
title: How to solve 'Invalid POST error' for cached forms submitted via AJAX in Drupal 7
---

In this post I will explain in detail a workaround for the 'Invalid POST error' for AJAX submissions of cached forms in Drupal 7.

-----

This article can be of help for you if:

1. Your Drupal 7 site contains forms submitted via AJAX
2. You have enabled page caching
3. Anonymous users should be able to submit those forms

Use cases where this bug might affect you:

* Login form which is submitted via AJAX
* Pages or content with an AJAX widget (for example, flagging or rating content by means of the [Flag](https://www.drupal.org/project/flag) or [Fivestar](https://www.drupal.org/project/fivestar) modules)
* Commerce products with an AJAX add to cart button (enabled by [Commerce Ajax Add to Cart](https://www.drupal.org/project/dc_ajax_add_cart), for instance)

If your Drupal 7 site contains these or similar use cases, it is likely to be affected by this bug, and many of your users won't be able to submit those forms.

### The root of the problem

This is a well-documented bug which has been reported in [Drupal.org](http://www.drupal.org) since 2012.

You can take a look at the original bug submission and the follow the work on the issue:

[drupal\_process\_form() deletes cached form + form_state despite still needed for later POSTs with enabled page caching](https://www.drupal.org/node/1694574)

Long story short, the cached form entry in the database is deleted after Drupal processes a form submission in `drupal_process_form`.  Any subsequent AJAX submissions of the same form will thus fail until the cached form record is regenerated in the database.

In my opinion this is a critical issue which deserves close attention.  AJAX requests in form submissions are the daily bread in many websites and so is page caching for anonymous users. Unfortunately, it doesn't look like this issue is going to be fixed for Drupal 7 any time soon, so I went ahead and worked out a way around it.

### The workaround

The idea behind this solution is quite simple. We just need to make sure that any given form actually exists in the cache before an user is able to submit it.  This can be done by injecting a tiny Javascript file which will ask the server to check whether our form is properly cached, recreating the form and rebuilding it's `form_id` in case it's not.

For the sake of simplicity, we will just assume our form is rendered by means of a custom field `field_custom` attached to a node.  Some bits of the following code should be adapted to each specific use case.

Let's go down to the nitty-gritty.

In order to put this solution into practice, we will need a custom module which implements `hook_menu` and `hook_form_FORM_ID_alter`.

The main module file: `rebuild_ajax_form.module`:

{% highlight php %}
<?php
/**
 * Implements hook_menu().
 *
 */
function rebuild_ajax_form_menu() {
  $items['rebuild_ajax_form'] = array(
    'page callback' => 'rebuild_ajax_form_rebuild_form',
    'page arguments' => array(1),
    'type' => MENU_CALLBACK,
    'access callback' => TRUE,
    'theme callback' => 'ajax_base_page_theme',
  );
	return $items;
}

function rebuild_ajax_form_rebuild_form($product_id) {
  if (isset($product_id) && isset($_POST['form_build_id'])) {
    // Get form from cache if it exists
    $form_state = form_state_defaults();
    $form = form_get_cache($_POST['form_build_id'], $form_state);
    if (!$form) {
      // Form not found, rebuild form and update form_id via AJAX
      // This implementation will depend entirely on your use case.
      // In our case, the form is rendered by means of 'field_custom',
      // so we reload the form by loading and rendering the field
      $node = node_load((int) $product_id);
      $output = field_view_field('node', $node, 'field_custom');
      $form = $output[0];
      // Keep track of the old `form_build_id`
      $form['#build_id_old'] = $_POST['form_build_id'];
      // Replace `form_build_id` with updated one
      $commands[] = ajax_command_update_build_id($form);
      $return = array(
        '#type' => 'ajax',
        '#commands' => $commands,
      );
      ajax_deliver($return);
    }
  }
}

/**
 * Implements hook_form_FORM_ID_alter().
 *
 */
function rebuild_ajax_form_form_your_custom_id_form_alter(&$form, &$form_state) {
  // Should this form include the JS file to fix the AJAX issue?
  if (isset($form_state['context']['entity_id'])) {
    $nid = $form_state['context']['entity_id'];
    drupal_add_js(array('rebuildAjaxForm' => array(
      'nid' => $nid,
      'form_build_id' => $form['#build_id'],
      ),
    ), 'setting');
    $basepath_mod = drupal_get_path('module', 'rebuild_ajax_form');
    drupal_add_js($basepath_mod . '/rebuild_ajax_form.js', array(
      'weight' => 100,
      ));
  }
}
{% endhighlight %}

The Javascript file: `rebuild_ajax_form.js`:

{% highlight js %}
/**
 * @file rebuild_ajax_form.js
 * Update form_build_id
 */
(function ($, Drupal, window, document, undefined) {
  Drupal.behaviors.rebuildAjaxForm = {
    attach: function (context, settings) {
   /**
     * @see https://www.deeson.co.uk/labs/trigger-drupal-managed-ajax-calls-any-time-drupal-7
     * Add an extra function to the Drupal ajax object
     * which allows us to trigger an ajax response without
     * an element that triggers it.
     */
      Drupal.ajax.prototype.specifiedResponse = function() {
        var ajax = this;
        // Do not perform another ajax command if one is in progress
        if (ajax.ajaxing) {
          return false;
        }
        try {
          $.ajax(ajax.options);
        }
        catch (err) {
          alert("An error occurred in: " + ajax.options.url);
          return false;
        }
        return false;
      };

      // Define a custom ajax action not associated with an element.
      var custom_settings = {};
      custom_settings.url = '/rebuild_ajax_form/' + settings.rebuildAjaxForm.nid;
      custom_settings.event = 'onload';
      custom_settings.keypress = false;
      custom_settings.prevent = false;
      custom_settings.submit = {form_build_id: settings.rebuildAjaxForm.form_build_id};
      Drupal.ajax['custom_ajax_action'] = new Drupal.ajax(null, $(document.body), custom_settings);

      $('form[id|="your-custom-id-form"]', context).once('rebuildAjaxForm', function(){
        Drupal.ajax['custom_ajax_action'].specifiedResponse();
      });
    }
  };
})(jQuery, Drupal, this, this.document);
{% endhighlight %}

Make sure to replace the `your-custom-id-form` instances with whatever the ID of your form is.

Your should also keep an eye on the `rebuild_ajax_form_rebuild_form` callback which will again depend on your specific use-case.

### Wrapping it up

I really hope you found this post useful.  As I mentioned above, the code needs to be tailored by your particular needs.  For example, if you want to rebuild the Login form, what you need to do is to reload the form by means of `drupal_get_form` instead of `node_load`, etc.

While this solution does the job, it could definitely be improved.  It could be turned into a full module which automatically detects AJAX submit buttons in forms, then figure out the best way to rebuild the form in each case.
