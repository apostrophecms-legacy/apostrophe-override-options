# ⛔️ **DEPRECATED** — do not use for new projects

See [our current docs](https://docs.apostrophecms.org/)

This module allows configurable overrides of the `getOption` method and Nunjucks helper of Apostrophe modules, based on:

* Editable fields of the widget being rendered, if we're rendering a widget, *or*
* Template-level options passed to the widget being rendered, if we're rendering a widget, *or*
* The current piece type or user-editable settings of the piece, when viewing the "show" template (permalink page for that piece), *or*
* The current page type or user-editable settings of a current page, or those of an ancestor in the page tree, *or*
* User-editable settings of the `apostrophe-global` module, *or*
* The options actually configured for the module (this is the default behavior of `getOption`).

In addition, if the `apostrophe-workflow` module is present, settings based on piece types and page types can be localized, and module-level default settings can be localized as well.

The end result is a general-purpose `module.getOption` helper that allows frontend developers to get the right thing easily in a template, and puts the responsibility of deciding what the right thing will be in a given context on the backend developer.

## Override syntax

```javascript
// in app.js

modules: {
  // Enable the feature
  'apostrophe-override-options': {}
}

// in lib/modules/landing-pages

module.exports = {
  name: 'landing-page',
  extend: 'apostrophe-custom-pages',
  addFields: [
    {
      name: 'analyticsEventId',
      type: 'string',
      label: 'Analytics Event ID',
      def: 'abc'
    }
  ],
  overrideOptions: {
    fixed: {
      // Note must begin with `apos.module-name` or `apos.moduleAlias`
      'apos.analytics-button-widgets.style': 'blue',
    },
    // only if `apostrophe-workflow` is present
    localized: {
      // Locales per `apostrophe-workflow` which must be in use
      'en': {
        'apos.analytics-button-widgets.style': 'purple',
      }
    },
    editable: {
      // `analyticsEventId` should be a schema field as seen above
      'apos.analytics-button-widgets.eventId': 'analyticsEventId'
    }
  }
};
```

The above code in the `landing-pages` module overrides what `getOption('style')` and `getOption('eventId')` will return in the `analytics-button-widgets` module.

The same technique may be used in a module that extends the `apostrophe-pieces` module, in which case it applies when the piece is being displayed on its own `show` page.

The technique may also be used in configuration of the `apostrophe-global` module, in which case it is most common to use the `editable` subproperty to make certain options of various modules overridable via the "global" admin bar item.

> As a convenience, you may choose to skip the `apos.moduleName` part if you are overriding an option of your own module, which is often done via `editable`.

## Appending and removing elements from arrays

This special syntax can be used to add and remove array elements from options:

```javascript
  // completely replaces the setting with a new array of one item
  'apos.analytics-button-widgets.eventIds': [ 'that-is-all' ]
  // appends to an array, which must already exist
  'apos.analytics-button-widgets.eventIds': { $append: [ 'at-the-end' ] }
  // prepends to an array, which must already exist
  'apos.analytics-button-widgets.eventIds': { $prepend: [ 'at-the-start' ] }
  // replace elements in array, only if a match is found
  'apos.analytics-button-widgets.eventIds': {
    $replace: [ { id: 42, value: 'newValue' } ],
    comparator: 'id'
  }
  // removes from an array, which must already be an array.
  // It is OK if the values removed are already gone
  'apos.analytics-button-widgets.eventIds': { $remove: [ 'this-one-goes-away' ] }
  // appends only if value not already present
  'apos.analytics-button-widgets.eventIds': { $appendUnique: [ 'last-if-missing' ] }
  // prepends only if value not already present
  'apos.analytics-button-widgets.eventIds': { $prependUnique: [ 'first-if-missing' ] }
  // merge and append the rest of non-matching elements
  'apos.analytics-button-widgets.eventIds': {
    $merge: [ { id: 42, value: 'changedValue' }, { id: 43, value: 'newValue' } ],
    comparator: 'id'
  }
```

### Customize the `comparator`

The comparator is available for `appendUnique`, `prependUnique`, `replace` and `remove` commands.

```javascript
// You can pass a custom function, as shown here, or a string containing
// a property name for a simple property comparison
'apos.analytics-button-widgets.eventIds': {
  $appendUnique: [ 'last-if-missing' ],
  comparator: function(a, b) { return a === b }
}
```

### Editable fields and `$append`, etc.

For `editable`, specify the field name as the value, i.e. `{ $append: 'fieldname' }`.

This does what you probably had in mind. If the field does not contain an array, it is treated as an array of one element as long as it is truthy or the *number* `0`. Otherwise it is treated as an empty array. So an empty field of type `string` does not change the array; a field with text appends that one value.

### Transforming options with functions

When using `fixed` override options, you may pass a function rather than a value. If you do so, your function will receive:

`(req, options, path, val)`

Where `req` is the request object (in which you may look for `req.data.bestPage`), `options` contains the options object of the relevant module as transformed by the operations processed so far, `path` is an array beginning with the first part of the key after the module name, and `val` is the existing value of the option, if any.

It's simpler than it sounds! Here's a typical example:

```javascript
overrideOptions: {
  fixed: {
    'apos.analytics-buttons.eventId': function(req, options, path, val) {
      // If we're on a show page for a piece, use its _id,
      // otherwise the configured default value for the module
      return req.data.piece ? req.data.piece._id : val;
    }
  }
}
```

## Overriding widget options with `editable`

Widget modules can use `editable` too. However they may *only* use `editable`, they may *only* override their own options, and the overrides are only seen by `module.getOption` calls made in `widget.html` or something invoked by it. Since widgets are not full-page experiences it does not make sense for them to override options of other modules.

## Localization of default options

This module also adds the ability to localize module-level default options directly in each module, when the `apostrophe-workflow` module is also present. This is a convenience that avoids the need to add a great number of `localized` overrides in `apostrophe-global`. The syntax is slightly different because the properties being modified belong to the same module.

```javascript
// in lib/modules/analytics-button-widgets/index.js

module.exports = {
  extend: 'apostrophe-widgets',
  name: 'analytics-button',
  label: 'Analytics Button',

  flavor: {
    mouthfeel: 'tangy',
    sweetness: 'very'
  },
  localized: {
    en: {
      'flavor.sweetness': 'very-en'
    }
  }
};
```

Note however that dot notation is still used for nested keys.

### Disabling page types based on the locale

This feature allows the disabling of page types based on the current locale, in conjunction with the `disabledTypes` option of the `apostrophe-pages` module. Just use `localized` as shown above to set `disabledTypes` to an array of types that should not be available in a given locale when creating new pages or changing the page type.

Note that while `localized` works here, option overrides that are dependent on the position within the page tree do not. This is because pages may appear at any point in the tree and it would be a false claim to try to restrict their schemas based on where they are "born" in the site.

## Inexact URL matches and the `show` pages of pieces

In a best effort to take URLs that contain additional components beyond the slug of the page into account, this module honors `req.data.bestPage` if `req.data.page` is not yet set.

## Visibility of late option modifications

If the original options of a module are modified after `pageServe` time, those changes will not be accessible at all to `getOption` calls made for that particular request when this module is in use. However, since module options are not request-specific, it would almost never make sense to modify them after app launch time.

## Visiblity of option settings when editing widgets

When you are editing a widget and click "save" and it re-renders, options inherited via the page tree are not visible. However when the widget renders later as part of a real visit to a page those options will be visible and will impact rendering as intended. There is a deeper issue in Apostrophe associated with this and it is under discussion.

## Timing concerns

For performance, this module computes its results just before `pageServe` methods are invoked. At this point, `req.data.bestPage` has been set, and widgets are about to be loaded.

Any invocation of `getOption` before this point will invoke the default implementation.

However, `req.data.piece` is not set until the `pageServe` process is already underway. To address this issue, this module recomputes its results when a `show` page is encountered. This means that the impact of the piece type or piece settings will be honored in `getOption` calls in templates, or in JavaScript code invoked by `pageBeforeSend`. It is, however, too late for `getOption` to be honored inside the `load` methods of widgets on the page.

You may optionally address this issue by passing this option to the `apostrophe-areas` module:

```javascript
modules: {
  'apostrophe-areas': {
    deferWidgetLoading: true
  }
}
```

With this change, areas invoke load methods for their widgets at the last possible moment, after all `pageBeforeSend` methods. This results in fewer database calls and also ensures that the impact of the current piece is visible in any `getOption` calls made by the widget loaders.

*This issue does not impact `widget.html` templates. If that is the only place you are making `getOption` calls for your widget you do not need to make this change.*

### Timing concerns with widget loaders for the `global` doc

Due to the middleware-based loading process for the `global` doc, `getOption` method calls by widget `load` methods for the `global` doc will not be able to see the impact of the current page in any scenario. Again, this impacts the load methods only, not `widget.html` files which **will** see it.

*TODO:* it may be possible to address this by further modifying `deferWidgetLoading` to defer the global doc to `pageBeforeSend` as well, which is invoked even if a page is being rendered via `sendPage`. This would need to be a new optional setting as developers invoking `renderPage` directly would not get widget loads this way.

## Limitations

### Options that cannot be overridden

Technically, `apos` itself is an option passed to each module. You cannot override properties of this object via the above syntax; an error will be reported. If Apostrophe allowed this the performance impact of deeply cloning the object to allow it to differ for each request would be prohibitive. Similarly you should avoid overriding properties of other large objects. Options that are configured in your modules using simple JSON-friendly data structures are much better candidates.

### Option cloning and supported data types

If at least one module alters an option via `overrideOptions` at any depth, all subproperties found beneath the same top-level key within the options for the module in question are recursively cloned. The performance impact is small if this module is only used to adjust simple "JSON-friendly" option data structures, and Date objects and functions are still included among the cloned properties. However, be aware of the [limitations of the Lodash cloneWith and cloneDeepWith functions](https://lodash.com/docs/4.17.4#cloneWith) (note that cloning functions is explicitly worked around in this module).
