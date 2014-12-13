WebL10n API Spec
=====================

This document outlines the design of the client-side localization API for 
HTML/DOM.

The initial approach is to base the API on the parts of l10n framework 
currently used in Firefox OS.

For the sake of this document, l20n file format will be used for l10n resource 
examples.

L10n Attributes
----------

The fundamental concept in WebL10n API is a set of two attributes that are bind 
to HTML Elements in order to mark them as localizable:

 - L10nID - is a unique identifier that binds the Element with its translation
 - L10nArgs - is a list of name-value pairs of variables that instrument the 
   translation.

In HTML the example may look like this:

    <h1 l10n-id="header" /> 

In a localization resource file for that language there will be 
a single localization entity with an ID "header" and translation content for the node:

    <header "My Header">

L10n Arguments are a nested lists of name-value pairs, where each value may be a name-value pair list as well. Those arguments will be usually provided by the developer at runtime similarly to how dataset works, except that we want to allow for nested lists.

Example:

    <h1 l10n-id="header" l10n-args='{"user":{"name":"John", "gender":"male"}}' />


    <header "Welcome, {{ $user.name }}">


Element Translation
----------

Element translation is composed of a value and a list of attribute translations.

A value may be either a Text Node or a DOM Fragment ([read about DOM Overlays](https://github.com/stasm/spec/blob/master/dom-overlays.markdown)).
Attributes contain the list of localizable attributes that will be overlayd on top of the Element to localize it.

An example is a localization entity like this:

    <header "Settings"
     ariaLabel: "Application Settings Header">

In the following example the localization entity "header" has a string value "Settings" and one attribute "aria-label" with a string value.

When L10nID is set on an Element, it should be localized using the value and attributes from the localization resources. When L10nID is removed, the engine should remove localization changes and display untranslated version of the element.
When L10nID or L10nAttrs are modified, the Element should be retranslated.

Javascript API
----------

From JavaScript is must be possible to access the L10nID, L10nArgs and the localized version of the DOM. Proposed API may look like this:

    var element = document.getElementById('foo');
    element.l10n.id; // L10nID string
    element.l10n.args; // JS Object with arguments
    element.l10n.args.user; // JS Object with {name: "John", gender: "male"}
    element.l10n.args.user.name; // a String
    
    element.l10n.root; // access to localized version of the DOM 

In order to faciliate runtime's library ability to provide localization of the Element, we need two functions.

One for registering a method that is being called when l10n attrs are being added/modified/removed. It may work similarly to Event Listener, or Mutation Observer. In the shim library we use Mutation Observers, but from within the platform we could probably use something more direct to avoid looping through the mutations. [code](https://github.com/mozilla-b2g/gaia/blob/e2a3e606675c346b6e6f35351a458040be599b09/shared/js/l10n.js#L1721-L1748)

Another for the library to provide the translation of the Element's content and localizable attributes to the platform.

Example of how it may look like:

    document.body.registerL10nListener(function(element) {
      mozL10n.formatEntity(element.l10n.id, element.l10n.args).then(function(l10nEntity) {
        element.setL10n(l10nEntity.value, l10nEntity.attrs);
      });
    });

Resource loading
----------

Similarly to CSS resources, localization resources must be loaded before we can localize Elements. We need to hook into the HTML engine to properly schedule Element localization to avoid redundant frame creation, layout and ensure that we properly avoid painting until the HTML is localized.

The example HTML header of a localizable document looks like this:

    <html>
      <head>
       <meta name="locales" content="en-US, fr, de">
       <meta name="default_locale" content="en-US">
       <link rel="localization" href="locales/example.{locale}.properties">
      </head>

Language negotiation will happen in mozL10n library, and will result in async XHR request for the resource file. Once the resource is loaded, mozL10n will be ready to localize Elements.

One idea is to provide a method for instrumenting Gecko to stop creating frames until l10n resources are loaded and one for enabling it again.
