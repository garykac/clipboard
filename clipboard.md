# Clipboard

DRAFT

Proposal for a modern, asynchronous clipboard interface.

## Proposal Summary

1. Introduce a new object: `navigator.clipboard`

2. Have `read()` and `write()` methods that return a `Promise`:

3. Add a new event listener to detect clipboard change events.


## Background

Since the introduction of
[designMode](https://developer.mozilla.org/en-US/docs/Web/API/Document/designMode),
there has been a web specification that supports clipboard access:
[document.execCommand()](https://w3c.github.io/editing/execCommand.html).

This API was originally not available to ordinary web pages in most browsers,
due to concerns about clobbering the clipboard and sniffing clipboard content.

Thus, the only reliable way to implement this feature across browsers until 2015 was
through Flash (e.g. [ZeroClipboard](http://zeroclipboard.org/)).
Anecdotally, for some sites this was the last remaining use of Flash that did not have
an open web API alternative.

However, as of 2015 all major browsers support document.execCommand(“copy”):

* Internet Explorer 9 (2011-04-14)
* [Chrome 42](https://googlechromereleases.blogspot.com/2015/04/stable-channel-update_14.html)
    (2015-04-14; [email thread](https://groups.google.com/a/chromium.org/d/msg/blink-dev/3QL6mAhC3Lw/rZ2S3YM-9XIJ),
    [bug](https://crbug.com/437908))
* [Opera 29](https://dev.opera.com/blog/opera-29/) (2015-04-28)
* [Firefox 41](https://www.mozilla.org/en-US/firefox/41.0/releasenotes/) (2015-09-22)
* Safari Technology Preview (2016-03-30; [announcement](https://webkit.org/blog/6017/introducing-safari-technology-preview/))

Internet Explorer allows “paste” as well as “copy” for plain text, but shows a permission prompt to the user for both.
All other browsers allow copying “text/plain” and/or “text/html” upon user gesture (without a user prompt).
Also, “cut” is currently supported everywhere “copy” is supported.

There is work on a [Clipboard API](https://www.w3.org/TR/clipboard-apis/) specification by
Hallvord R. M. Steen of Mozilla. However, this API is strongly rooted in the assumption
that it should be based on document.execCommand(), and Hallvord has even
[started a thread](https://lists.w3.org/Archives/Public/public-webapps/2015JulSep/0235.html)
questioning whether this is appropriate in the long term.


## Motivation

Web apps commonly offer a convenient “copy to clipboard” button. A few apps built on the web platform also have advanced use cases that benefit from more clipboard access. In 
particular:

* Some apps want to be able to copy a selection without requiring specific user interactions to copy/select the content.

* Some apps (e.g. rich document editors) want to read the clipboard without requiring the user to initiate an OS “paste” action every time.

* Some apps (e.g. remote desktop) want to receive updates about clipboard changes.


## Proposal

This section provides more details about the proposal.

#### `navigator.clipboard`

Note: Having this on the `navigator` means that it would be accessible
from workers.

#### `read()` and `write()` methods

Return a `Promise`.

* `write()` would take a MimeTypeObject and return a Promise<>
* `read()` would return a Promise<MimeTypeObject>

Where a MimeTypeObject is a dict of mimetype-string:values

Possibly have convenience methods like writeText(/* string */)

#### Event listener for clipboard change events


## Example Usage

Write example:

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

Read example:

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
    
Detect clipboard change example:

```javascript
/**
 * @param {ClipboardEvent}
 */
function listener(clipboardEvent) {
    // Do stuff with clipboardEvent.clipboardData
}

navigator.clipboard.addEventListener(“clipboardchange”, listener);
```

## Asynchronous vs Event-driven

The current [Clipboard API](https://www.w3.org/TR/clipboard-apis/) is event
driven, which is 


## Potential for Abuse

There are a few avenues for abuse that are not specific to this proposal,
but are applicable to any API that provides clipboard access.

One of the abuse vectors in particular, pasting images, is part of the 
motivation for this proposal. In order to clean up malicious images,
they would need to be decoded and it is not appropriate to do this on
the main thread (large images could lock the browser).

#### Reading from the clipboard

Sniffing the clipboard contents. Of concern not just because of the possibility
of PII, but also because it is not uncommon for users to copy/paste passwords
(e.g., from a password manager to a website).

Consider: Can we respect having clipboard contents marked as 'sensitive'? This
would require OS support, but is apparently possible on OSX.

#### Writing to the clipboard

Inject malicious content onto the clipboard.

###### Pasting Text

Malicious text can be in the form of commands (e.g., 'rm -rf /\n') or
script ([Self-XSS](https://en.wikipedia.org/wiki/Self-XSS)). 

###### Pasting Images

Images can be crafted to exploit bugs in the image-handling code, so they
need to be scrubbed as well.


## Mitigating Abuse

To mitigate against potential abuse of this feature, we have a few obvious
options, some of which can be combined.

#### Do Nothing

Since clipboard options are considered to be basic functionality by most
users, doing nothing to mitigate abuse is certainly an option:

Pros:

* Cut/copy/paste work as the user expects.

Cons:

* Users get no indication that page interacted with the clipboard, so if
    they may be surprised to find things there they didn't explicitly
    put there (possibly overwriting something they wanted to keep).

#### Require a user gesture

Pros:

* Harder for malicious code to trigger since it would require some social
    engineering as well (to get the user interaction).

Cons:

* This would (by design) prevent purely programmatic access to the clipboard.

#### Only allow from front tab

Pros:

* This behavior is probably what the user expects. At least for cut/copy/paste.

Cons:

* It would prevent the creation of a clipboard history app. (Although I think
    this might be a "Pro" since a clipboard history app is probably better
    implemented as a native app.)

#### Pop-up Notifications

A post-facto notification similar to what is done for fullscreen. Display
something like: "New data pasted to clipboard" or "Data read from clipboard".

Pros:

* Low-friction for the user, while still notifying them of the clipboard
access.

Cons:

* ???

#### Permission 

Ask the user for permission, either at page load (as is done for cookies in 
Europe) or when the feature is first used.

Pros:

* Users need to explicitly opt-in, so UA's can throw up their hands and claim
    "Hey, it's not our fault" if something bad happens. ^_^

Cons: 

* Users dislike these interruptions to their workflow and tend not to read
    and understand the implications. Especially with standard operations like
    cut/copy/paste that don't sound scary.
* Users have been trained to ignore these messages.


## Current API

What happens with `document.execCommand()` and the current clipboard operations
that depend on it. 


## Acknowledgements

Thanks to the following people for the discussions that lead to the creation
of this proposal:

Daniel Cheng (Google),
Lucas Garron (Google),
Gary Kacmarcik (Google),
Hallvord R. M. Steen (Mozilla),


## References

[Clipboard API](https://www.w3.org/TR/clipboard-apis/)
