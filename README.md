This repository illustrates a memory leak from iframes in Chromium-based browsers.

# The leak

The leak occurs because V8's Console API stores strong references to objects in the 
Console, even when DevTools isn't open. The leak occurs here in `iframe.html`, but 
can be observed even when the iframe is unloaded, which is particularly egregious.

In this minimal repro, `iframe.html` causes a large object to be allocated and 
attached to the global, which simulates memory pressure from the internal iframe. 
The iframe is then unmounted and discarded from the application's perspective. 
However, if the console message is output (regardless of whether DevTools are open),
the instance of the `TypeError` thrown by this iframe will be retained by the 
V8ConsoleMessage object. Consequently, the error's prototype chain will be retained,
which will ultimately keep a strong reference to the `window` global corresponding
to the iframe.

If the iframe is removed before the error message can be logged, the heap will 
still reflect the size of the array; however, manually triggering a garbage 
collection will cause this to be cleared. But, once the console message is emitted,
the strong reference will stay until the user clicks the "Clear console" button
in DevTools.

## Presence in Node?

This could potentially cause a memory leak in Node, since the retainer actually 
lives within the V8 runtime's Inspector implementation. 

## How to fix?

It seems as if firm objects should not be retained when the Inspector is not 
active. Rather, enough information to generate an ObjectPreview should be generated,
or a serialized instance of the error.
