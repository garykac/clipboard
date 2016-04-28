# Clipboard

DRAFT

Proposal for a modern, asynchronous clipboard interface.

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

1. Introduce a new object: `navigator.clipboard`

2. Have `read()` and `write()` methods that return a `Promise`:

    * `write()` would take a MimeTypeObject and return a Promise<>
    * `read()` would return a Promise<MimeTypeObject>

    Where a MimeTypeObject is a dict of mimetype-string:values

    Possibly have convenience methods like writeText(/* string */)

3. Add a new event listener to detect clipboard change events.


## Example Usage

Write example:

    // Markup with specified MIME type.
    navigator.clipboard.write({
        “text/html”: “<b>Howdy</b>, partner!”
    });

    // Multiple MIME types.
    navigator.clipboard.write({
        “text/plain”: “Howdy, partner!”,
        “text/html”: “<b>Howdy</b>, partner!”
    });

    // Use the Promise outcome to perform an action.
    navigator.clipboard.write(“text”).then(function() {
        console.log(“Copied successfully!”);
    }, function() {
        console.error(“Unable to write. :-(”);
    });

Read example:

    // Common use case: pasting a string
    navigator.clipboard.read().then(function(clipboardData) {
        if (“text/plain” in clipboardData) {
            console.log(“Your string:”, clipboardData[“text/plain”])
        } else {
            console.error(“No string for you!”);
        }
    })
    
    // General case: pasting a string
    navigator.clipboard.read().then(function(clipboardData) {
        // Do stuff with clipboardData
    })

Detect clipboard change example:

    /**
     * @param {ClipboardEvent}
     */
    function listener(clipboardEvent) {
        // Do stuff with clipboardEvent.clipboardData
    }

    navigator.clipboard.addEventListener(“change”, listener);


## Permission


## Acknowledgements

Thanks to the following people for the discussions that lead to the creation
of this proposal:

***

## References

***
