# Clipboard

DRAFT

Proposal for a modern, asynchronous clipboard interface.

## Proposal Summary

1. Introduce a new object: `navigator.clipboard`

2. Have `read()` and `write()` methods that return a `Promise`

3. Add a new event listener to detect clipboard change events.

#### Goals

* API for safe, asynchronous clipboard access

#### Non-Goals

* Eliminate all usage of `document.execCommand()`


## Background

An API for clipboard actions (cut, copy and paste) has long been desired for webpages,
but agreement has been slow because of various security and usability concerns with
the feature. In summary, giving web pages access to the clipboard introduces problems
with clobbering (overwriting) and sniffing (surreptitiously reading) the clipboard.

This lack of a consistent API has forced web developers who need this feature to rely
on alternate methods (e.g., Flash, via [ZeroClipboard](http://zeroclipboard.org/)) which
simply trades one set of problems for another.

Recently, however, all major browsers have converged on support for clipboard access
using [`document.execCommand()`](https://w3c.github.io/editing/execCommand.html):

                   | cut  | copy | paste
    -------------- | ---- | ---- | -----
    IE &sup1;      |   9  |   9  |    9
    Chrome &sup2;  |  43  |  43  |   no
    Firefox &sup2; |  41  |  41  |    ?
    Opera &sup2;   |  29  |  29  |   29
    Safari &sup2;  | yes &sup3;  |  yes &sup3;  |   ?
    Edge &sup2;  |  12  | 12   |  14 <sup>4</sup>

&sup1; A permission prompt to the user when the feature is used.

&sup2; Only permitted when executing code in response to a user action or gesture.

&sup3; The Safari Technology Preview ([announced on 2016-Mar-30](https://webkit.org/blog/6017/introducing-safari-technology-preview/))
has support for cut and copy.

<sup>4</sup> Edge 14 supports paste only through an offline-managed Allow List.

However, even with this increased support, there are still many inconsistencies across
browser implementations. See
[this blog post](https://www.lucidchart.com/techblog/2014/12/02/definitive-guide-copying-pasting-javascript/)
for a description of the problems Javascript programmers face, and note the
existence of various Javascript
[clipboard](https://github.com/lgarron/clipboard.js)
[libraries](https://github.com/zenorocha/clipboard.js)
to help address these issues.

As noted earlier, the current [Clipboard API](https://www.w3.org/TR/clipboard-apis/)
specification that the browsers are implementing assumes that the API should be
based on `document.execCommand()`. However, there have recently been
[discussions on public-webapps](https://lists.w3.org/Archives/Public/public-webapps/2015JulSep/0235.html)
questioning whether this is appropriate in the long term.


## Motivation

The specific issues that motivate this proposal are:

* The current model is synchronous, so it blocks the main page.
  * This makes permission prompts more irritating (since the page is blocked).
  * This prevents sanitizing data types (like images) that need to be transcoded.
    * Transcoding is sometimes required to guard against exploits in external parsers.
    * It's not reasonable to block page while large images are being sanitized.

* The code required for programmatic clipboard actions is... bizarre:
  * To modify the clipboard, you need to add an event listener for 'copy'. This listener will
    fire when you call `execCommand('copy')` and also for any user initiated copies.

* There is a need for a notification event when the clipboard is mutated.
   * Apps that synchronize the clipboard (like remote access apps) need this to
     know when to send the updated clipboard contents to the secondary
     system.

* `execCommand` is old API originally designed for editing the DOM and it has a large number of
  [interoperability bugs](https://github.com/guardian/scribe/blob/master/BROWSERINCONSISTENCIES.md).
  The latest version of the
  [execCommand spec](https://w3c.github.io/editing/execCommand.html)
  states that it is incomplete and not expected to advance beyond draft status.

* In practice, many developers load a library to use `execCommand` properly. That shouldn't be necessary for something as basic clipboard cut/copy/paste.

And finally,

* Make it easier to support alternate permission models.
   * For example, to enable web apps to copy to the clipboard without requiring a user gesture.
   * Enable use of the Permissions API or a consent dialog that doesn't block the browser.


## Proposal

This section provides more details about the proposal.

#### `navigator.clipboard`

The main clipboard object is part of the `navigator` because it is a
"system" level object that is not tied to the current window or document.

Note: Having this on the `navigator` means that it would be accessible
from workers.

#### `read()` and `write()` methods

These methods would return a `Promise` that is resolved when the reading or
writing operation is complete.

* `write()` would take a `DataTransfer` and return a `Promise<>`
* `read()` would return a `Promise<DataTransfer>`

It is also desireable to have some convenience methods like `writeText()`
or `writeHtml()`.

#### Event listener for clipboard change events

This event would fire whenever the clipboard contents are changed. If the
clipboard contents are changed outside the browser, then this event would
fire when the browser regains focus.


## Example Usage

Example of writing to the clipboard:

```javascript
  // Using convenience function to write a specific MIME type.
  navigator.clipboard.writeText("Howdy, partner!");

  // Multiple MIME types.
  var data = new DataTransfer();
  data.items.add("text/plain", "Howdy, partner!");
  data.items.add("text/html", "<b>Howdy</b>, partner!");
  navigator.clipboard.write(data);

  // Use the Promise outcome to perform an action.
  navigator.clipboard.writeText("some text").then(function() {
      console.log(“Copied to clipboard successfully!”);
  }, function() {
      console.error(“Unable to write to clipboard. :-(”);
  });
```

Example of reading from the clipboard:

```javascript
  // Reading data from the clipboard.
  navigator.clipboard.read().then(function(data) {
      for (var i = 0; i < data.items.length; i++) {
          if (data.items[i].type == "text/plain") {
              console.log(“Your string:”, data.items[i].getAs(“text/plain”))
          } else {
              console.error(“No text/plain data on clipboard.”);
          }
      }
  })
```

Example of detecting clipboard changes:

```javascript
  function listener(event) {
      // Do stuff with navigator.clipboard
  }

  navigator.clipboard.addEventListener(“clipboardchange”, listener);
```

## Current Clipboard API

The current Clipboard API describes events that are fired when either:

1. the user selects one of the standard clipboard actions via the browser's UI
    or keyboard shortcuts (these are "trusted" events), or
2. javascript code sends one of these events (in which case, they are
    "synthetic" and "untrusted").

With this proposal, these events would still be present, but the recommended way
to access the clipboard would be through the Promise-based APIs rather than
via `execCommand` (although the current `execCommand`-based API would stick
around for compatibility reasons).

The current model of requiring some sort of permission or opt-in before allowing
untrusted access to the clipboard would be retained.

Basically, the intent (if there's agreement) is to merge this into the
current Clipboard API document so that all the clipboard-related APIs live
in the same document.

Note that with this proposal, the `ClipboardEvent` type could be replaced with
a simple `Event` and all clipboard access could be through the
`navigator.clipboard` object.

Detect clipboard change example:

```javascript
  function listener(event) {
      navigator.clipboard.read().then(function(data) {
          // Do something with clipboard data.
      });
  }

  navigator.clipboard.addEventListener(“copy”, listener);
```


## Potential for Abuse

There are a few avenues for abuse that are not specific to this proposal,
but are applicable to any API that provides clipboard access.

It is one of these abuse vectors in particular, pasting images, that motivates
this proposal. In order to clean up malicious images,
they would need to be decoded and it is not appropriate to do this on
the main thread (large images could lock the browser while the image is
being processed).

#### Reading from the clipboard

Sniffing the clipboard contents. Of concern not just because of the possibility
of PII, but also because it is not uncommon for users to copy/paste passwords
(e.g., from a password manager to a website).

#### Writing to the clipboard

Inject malicious content onto the clipboard.

Note, that it is already possible to clobber the clipboard contents:

```javascript
  document.addEventListener('copy', function(e) {
    // Modify the document selection or call e.clipboardData.setData()
  }
```

###### Pasting Text

Malicious text can be in the form of commands (e.g., 'rm -rf /\n') or
script ([Self-XSS](https://en.wikipedia.org/wiki/Self-XSS)).

###### Pasting Images

Images can be crafted to exploit bugs in the image-handling code, so they
need to be scrubbed as well. Transcoding large images can be computationally
expensive, so care must be taken to avoid processing them on the main thread.


#### Mitigating Abuse

Currently, user agents mitigate abuse by untrusted actions by either requiring
a user gesture (e.g., clicking on a button) or with a permission dialog.
These approaches suffer from the following issues:

**User gestures** provide defense against "drive-by" clipboard access, but the
user receives no notifications if the clipboard is accessed as part of an
unrelated user gesture. An example or this would be tricking user to click on
innocous "OK" button and then silently writing to the clipboard. In this
situation, the user grants no permission and receives no notification.

Pop-up **permission dialogs** can be problematic because clipboard events are
cancelable, so the browser needs to wait until the event handler is done (to know
whether or not it was canceled) before continuing. If the event handler
directly calls `execCommand` (which is also synchronous), then the browser is
blocked until the command (including any permission dialogs) is complete.
Note that replacing `execCommand` with an asychronous clipboard API would
make these permission dialogs more user-friendly.

For this feature, we should consider some combination of the following:

* Require a user gesture. To protect against drive-by access, although this may
    not be necessary with the right set of permissions.
* Only allow clipboard access from code running in the front tab.
* Pop-up Notifications. A post-facto notification similar to what is done for
    fullscreen. Display something like: "New data pasted to clipboard" or "Data
    read from clipboard".
* Permission Dialog. With an async clipboard API, this would be more
    acceptable since it wouldn't block the main process.
* [Permissions API](https://www.w3.org/TR/permissions/). Add a "clipboard"
    name to the registry


## MIME Types

This proposal is motivated by the desire to support image types in a safe
manner, but there is a desire to handle all types safely.

To do: Need to agree on set of required mimetypes. Can we safely allow any
mimetype and binary data? What alternatives are there for application-specific
data on the clipboard? `DataTransfer` would need to be updated for this since
it currently only supports text and files.


## Acknowledgements

Thanks to the following people for the discussions that lead to the creation
of this proposal:

Daniel Cheng (Google),
Lucas Garron (Google),
Gary Kacmarcik (Google),
Hallvord R. M. Steen (Mozilla),


## References

[Clipboard API](https://www.w3.org/TR/clipboard-apis/)

