---
id: ServerSideRendering
title: Server-Side Rendering
---

In some cases, you might want to include (static) HTML for a notebook on the HTML page rendered by your server so it can be shown even before the necessary JavaScript code is loaded and executed. This reduces the time until the user sees the notebook, and it also helps search engines to index your notebook content. Once your JavaScript code executes, you can use the function [WolframNotebookEmbedder.embed](./LibraryInterface.md) as usual to embed the notebook into the same container node, and it will transition seamlessly from the static view to "live" (interactive) rendering. The same happens on [www.wolframcloud.com](https://www.wolframcloud.com) as well.

Static HTML is generated ("pre-rendered") for each cloud notebook whenever it is created or changed. You can access it through an HTTP API, by making a `GET` request to

    https://www.wolframcloud.com/statichtml/{path}
    
where `{path}` is the notebook's cloud object path, which might be a UUID (e.g. `4beadfbb-84dd-4b26-87b6-bcd30b9abd65` for a cloud object at `https://www.wolframcloud.com/obj/4beadfbb-84dd-4b26-87b6-bcd30b9abd65`) or a user base URL + folder/file name (e.g. `myname/foo/bar.nb` for a cloud object at `https://www.wolframcloud.com/obj/myname/foo/bar.nb`).

This API returns a piece of static HTML that can be included right on your page by your server-side code. You might want to cache the response from the `/statichtml` API on your server, so you don't have to request it whenever a user visits your website.

A typical sequence of events would be:

1. A user makes a request to your server to open your website, which includes an embedded notebook.
2. Your server makes a request to `https://www.wolframcloud.com/statichtml/{path}` to fetch the static HTML for the embedded notebook, or it uses an existing cache for that static HTML.
3. Your server returns an HTML page to the user, which includes the static HTML of the notebook inside the HTML element where the notebook is supposed to be embedded.
4. In your JavaScript code, you call `WolframNotebookEmbedder.embed` to embed the notebook into the same HTML element, which causes the notebook to "live-render" on the client side. See [Notebook Loading Phases](./NotebookLoadingPhases.md) for more information about what happens then.

To summarize:

                 Wolfram Cloud
                     ^   |
                     |   | static HTML for notebook
     /statichtml API |   |
                     |   v
                  Your server
                     ^   |
                     |   | your website including static notebook HTML
             request |   | and JS code using WolframNotebookEmbedder
                     |   v
              User of your website
      

If you do not have active code running on your server but just a static HTML page, you could also use this API to generate HTML for the notebook once and then include that right in your static HTML. The downsides of this approach are that the HTML will not automatically update if the notebook is changed, and it will not be kept in sync with newer Wolfram Cloud versions (potentially resulting in rendering inconsistencies).

## Waiting for HTML to be available

Sometimes, there might be no HTML for a notebook available yet, e.g. when it has just been edited and the cloud server has not had a chance to pre-render it yet. In that case, the API will return an empty response. However, you can tell the API to wait up to a certain time and return static HTML once it becomes available. That's what the query parameter `maxwaitmillis` is for, e.g.

    https://www.wolframcloud.com/statichtml/4beadfbb-84dd-4b26-87b6-bcd30b9abd65?maxwaitmillis=500
    
would wait up to 500 milliseconds in case there is no HTML available yet and return once it becomes available. If the cloud server has not finished generating HTML after half a second, the API will return an empty response.

The maximum accepted value for `maxwaitmillis` is 20,000 (20 seconds). Typically, using rather low values (not longer than 1 second) is recommended, since this would delay the initial response from your server (while it is waiting for the cloud server to return HTML) and thus the loading of the whole page.

## Specifying a notebook width

Parts of the static HTML might depend on the page width, e.g. to determine where to line-wrap code cells. Since the page width is not usually known on the server, it can only assume a certain notebook width and return HTML for that, which will cause text to reflow on the client side (once the actual page width is known and "live" rendering kicks in). If you do know the notebook width in advance (e.g. because you are embedding the notebook at a fixed width), you can request HTML for that particular width using the `width` parameter to the API, e.g. 

    https://www.wolframcloud.com/statichtml/4beadfbb-84dd-4b26-87b6-bcd30b9abd65?width=700
    
would request HTML for a notebook width of 700 pixels. There is no guarantee that you get HTML for *exactly* that width, but the API will choose the best match out of the available pre-rendered HTML.
