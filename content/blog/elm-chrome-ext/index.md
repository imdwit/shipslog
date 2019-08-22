---
title: How I built a Chrome extension with Elm
path: "/elm-chrome-ext"
date: "2019-08-22T21:28:24.843Z"
description: "Building a chrome extension with Elm and a bit of JavaScript"
---

First let’s start with a bit of background.

I got a job as an Elm developer winter 2018.

At the time I didn’t know Elm so I had to learn in a hurry.

I had the idea for a chrome extension than can update the images I use on a website or app. But to go along with the chrome extension I had to build a whole dashboard (which I’ll write more about that later).

Over the next weeks and months I built and rebuilt the app I call [Vertol](https://tryvertol.com)
I learned how to build an Elm application by reverse engineering our app at work, as well as looking over @TODO[LINK] Richard Feldmans SPA app (v18) and a number of other tutorials.

Researching "how to build a chrome extension in Elm" didn’t yield much besides [danneu/elm-chrome-extension: a chrome extension composed of three elm apps that synchronize a single model](https://github.com/danneu/elm-chrome-extension) . Looking at how danneu structured their extension helped a lot with getting something set up. I took a lot of inspiration from danneu. I ended up needing all 3 main components of a chrome extension. A background script, popup script and content script.

My directory structure is a little different than danneu though.
[TODO SCREENSHOT]
The source code for the chrome extension shares code with Vertol’s dashboard. So to make things easy I just have the whole source for the extension inside the `src` folder, `manifest.json` and all, and I just simply `gitignore` the compiled output. Then a build script [TODO SCRIPT] outputs it right back into the src folder (¯\_(ツ)\_/¯).

#### How the extension is built:

- `src/extension/`

  - `/bg/bg.js`

        - handles connecting the `Background.elm` to the chrome runtime

  - `/popup/init.popup.js`

        - handles log in
        - sets up message handlers to connect the `Popup.elm` model to the chrome runtime
        - when a user opens the popup from the chrome extension it asks for the `Popup` model from `Background.elm`

  - `/content/content.js` - responsible for injecting some css and adding event handlers to enable drag and drop of new images - when an image is selected it is base64 encoded and passed to `Background` to cache in what I call a ‘staging’ area - when a user hits save the image is uploaded to s3 and updated ion the DB

* `src/Background.elm`

  `Background.elm` and `bg.js` are the brain, it persists the state for Popup.model, and handles the xhr requests. Passing messages back and forth between the popup, as well as the content scripts.

```
-- Background model
type alias Model =
    { popUp : PopUp.Model
    , hostname : String
    , projects : List Project
    , staging : Dict String Asset.Preview
    }
```

- `src/Popup.elm`

  Popup lets you log in (more soon), and passes that data to background, so it can check if you own this site/app to enable the content scripts.
  Being that chrome extension popups are effectively stateless, every time you open the popup it has to reinit the app. So storing the state of `Popup`’s model in `Background` makes it super simple to pass the model in as flags. When Popups model changes, it gets passed to Background so the next time the popup is open it’s initialized with the right state.

After you login and it registers the fact that the website you’re on is one of your projects. The content scripts wire up the event handlers for updating your images. It registers drop events on the `<img/>` tags to enable drag and drop. When an image file is selected it is then base64 encoded and passed to `Background` where it can remain cached in a

`{staging : Dict String Asset.Preview}`

So in the event a user accidentally refreshes, or navigates away from the page, the user doesn’t have to reselect new images, and they can just hit save to upload the replacements.When a user hits ‘save’, Background requests a signed url from the api for a PUT request for the S3 bucket where images are stored. The image is uploaded, and progress events are sent to the content script to increment a progress bar on the image.

`Popup` and `Background` are essentially independent elm applications that send messages out through _elm_ ports to the chrome runtime’s ports.

- `src/extension/content/content.js`

  The content script first injects some css to style the buttons and progress bars for new images. It uses `chrome.extension.sendMessage` to send messages to `chrome.runtime.onMessage.addListener` in `bg.js`. For instance there’s an `addFileToStaging` message that puts the base64 encoded image into `Background.elm`’s staging `Dict`.
  When a new image is selected the image is replaced inline with a base64 encoded version, but not yet saved. The user has the option to undo selecting that image, which puts the image back how they found it. Or they can hit ‘save’ which dispatches an `upload` event to `bg.js` which then uploads the image to S3. Content then receives progress events to update the progress bar’s value to indicate to the user something is happening.

Thanks for checking out this quick tour of my chrome extension. I learned a ton making it and hope to improve upon it in the near future. If you have any questions feel free to reach out to me on twitter at [TODO TWITTER LINK]
