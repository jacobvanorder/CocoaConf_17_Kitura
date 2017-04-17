# Kitura

## Overview

We will be making a *VERY* basic web backend that takes information from a client (an iOS app), stores it, and vends JSON responses.

We will not be covering front-end web development or anything that should be shipped in production. This is merely for if you want to have a hobby project or do a web server for a work project.

I'll be releasing these slides and source code on Github so you should have a chance to focus on what I'm saying instead of scribbling down notes.

## But First…

### What is Kitura

* Web server framework written in Swift
* IBM
* Open Source
* Inspired by Express.js which is inspired by Sinatra
* Alternatives
    * Vapor
    * Perfect
* What makes Kitura different than those two other frameworks is:
    - IBM is directly contributing to Foundation on Swift and is trying to make Kitura use only Foundation-based components.
    - They have a focus on backwards compatibility and obscure enterprise components
* They have a Slack channel and respond to issues on Github

### Swift Package Manager

* Apple
* Command Line Generation
* Creates file structure and basics
* Can create Xcode Project

### HTTP

#### Request

*  URL structure
    - scheme
    - path
    - port
    - path
    - query
* Verbs
    - GET
    - POST
    - PUT
    - DELETE
* Headers
    - Various types
    - key values
* Body
    - Data

#### Response

* Status code
* headers
* body

## Example App

### Baseball Cards!

* Application to store your baseball cards
* Functionality
    * Create baseball cards
    * See all baseball cards
    * See *your* baseball cards
    * Delete baseball cards
* How?
    * Mongo DB for database
    * Twitter Authentication for Auth
    * Kitura to pull it all together

## Let's Get Started

1. Create a directory you'd like the project to be called. In our case, “BaseballCards”.
2. `cd` into that directory.
3. Kick off Swift Package Manager with `swift package init --type executable`

### Resulting File Structure

BaseballCards
    ├── Package.swift
    ├── Package.pins
    ├── Sources
    │   └── main.swift
    └── Tests

* At this point, try to run `swift build`.
* We can run that by going to `.build/debug/BaseballCards`
* But where did it get the instructions to display “Hello World"?

## Dependencies and Swift Package Manager

* Because SPM is a dependency manager, we need to load up some dependencies!
* The format looks like:

```
import PackageDescription

let package = Package(
    name: "BaseballCards"
)
```

We will need to:

1. Add a key of `dependencies` with an array as the value.
1. Add a `.Package` enum with a `url` and `majorVersion`. You can also specify:
    * Minor version
    * specific version
[!] One word of caution, commit pinning isn't coming to SPM until Swift 4 if not later so our dependencies have to be in lock step with each other.

The result will be:

```
import PackageDescription

let package = Package(
    name: "BaseballCards",
    dependencies: [
        .Package(url: "https://github.com/IBM-Swift/Kitura.git", majorVersion: 1)
    ]
)
```

After you do this, run `swift build` and at this point, Swift Package Manager will fetch the dependencies you specified and build your binary. You can see where it is located when it says `linking .build/debug/BaseballCards` after you build. You run it by typing `.build/debug/BaseballCards` which you'll get a simple “Hello World” response. But where did that come from? Our main entry point into our executable is `main.swift` which is located right in the `Sources` folder.

### A Note about Linux

* The wonderful thing about Swift being open-source is that it can run on Linux.
* This means you can run it on:
    - a Raspberry Pi
    - cloud server somewhere
    - You can even run it in a Docker instance which IBM has created and is available.
*
* If you do write something you're deploy on Linux, you're going to want these tools available because there are still Foundation classes that are different or missing on Linux.
* Think of this another Unit Test that you'll have to check while you write your Swift on a Mac.
* All of this is out of scope for this 45 minute talk so I'm going to keep it Mac-only since we are largely iOS and Mac developers here.
* **!!Example? NSError? NSDateFormatter?**
* Code for separating out:
```
#if os(macOS) || os(iOS) || os(tvOS) || os(watchOS)
	// Your Darwin-based code here
#elseif os(Linux)
    // Your Linux-based code here
#endif
```

### Xcode

* With that out of the way, let's talk about how to actually edit this code within Xcode.
* Luckily Swift Package Manager has a command line instruction to generate an `.xcodeproj` file.
* `swift package generate-xcodeproj`
* At this point, SPM will go out and fetch the packages you have as dependencies if you haven't done this already
* This will create a Xcode project with two schemes:
    - A module for your project
    - The executable you can run.
* Annoyances/Notes
    - By default, the project is not testable. This is remedied by selecting `YES` for the `Enable Testability` option within the executable's Build Settings
    - If you add a dependency, you have to redo this process
    - The resulting Xcode project will have any changes you made to the Xcode project blown away
        - You will have to reselect the executable scheme each time
        - `Enable Testability` will be set to `NO`
    - This xcodeproj file is placed under .gitignore by swift package manager
* But… You can Build and Run!
* And code completion!

### Hello World (1)

1. First thing we do is `import Kitura`
2. Next, we instantiate a `Router` which is part of the `Kitura` library. This will route our incoming traffic.
3. We next set up a `get` at the path of `/` which is to say the root. This method has a variadic parameter of a `RouterHandler` completion closure.
4. This closure passes in a `RouterRequest`, `RouterResponse`, and a closure that tells the router to move on to the next `RouterHandler`.
5. Within this closure, we have the response send a string and call the `next` closure.
6. Finally, we attach the router to Kitura and start Kitura.

With these steps in place, we make sure our executable scheme is selected and Build and Run. Mac OS X will ask if we want to allow listing on that port.
If we use Postman to make a `GET` to `localhost:8090`, we should get a response of “Hello, world!”.

### Card Model (2) (XCODE)

Next, we are going to create a model object for the card that contains a couple properties.

* player name
* team names
* year of the card
* card number
* company who produced the card
* an identifier that we'll use later
* an userID which we'll use later.

Pretty standard stuff here.

We'll also add a `json(from:with:from:)` and `dictionary(from:with:from:)` convenience methods for later use.

**NOTE** We have to move our that are in the `Sources` directory into a directory called `BaseballCards`. If we don't when we generate an Xcode project, we'll get the error `error: the package has an unsupported layout, unexpected source file(s) found: …/Sources/main.swift`

### C.R.U.D V1 (3d)

#### `PUT` `v1/card` (3a) (XCode)

We need the ability to create a card.

1. We need to import SwiftyJSON and Foundation.
2. We'll create an `array` of `BaseballCards` for storage.
3. This endpoint will use a body parser which will do the job of parsing our request body into a SwiftyJSON object
4. We'll create a `PUT` endpoint at `v1/card`
5. Within the `RouterHandler`, we'll check:
    - We have a Content-Type in the headers and that it is `application/json`.
    - We'll make sure the request has a body.
    - If not, we send a `badRequest` response.
6. We'll next check to see if the body is SwiftyJSON or else send `unsupportedMediaType`.
7. Next, we'll create the model struct from all of the data in the SwiftyJSON data.
8. We'll append that to our storage array.
9. Finally, we'll send the card's id so the client can refer to it later.

### `GET` /v1/card/:id (3b) (XCODE)

1. We'll create an endpoint for `GET` with a special `:id` at the end. This signifies that there is going to be a variable in the URL that will be named `id`.
2. We pluck out that variable value and create a local variable. If it isn't found, we send a `badRequest`.
3. We then use the `filter(_:)` method on our storage array and gather the first card that has the same `id`.
4. We will send along that card as a `SwiftyJSON` payload by translating it to a dictionary first.
5. If no card is found, we send `notFound` response.

### `POST` /v1/card/:id (3c) (XCODE)

1. Like before, we will be using the body parser to parse the request body.
2. We'll create a new endpoint for `POST` to `v1/card/:id` like our `GET`
3. We'll be checking content-type, body, and if the request has an `id`.
4. Like before, we'll get our `SwiftyJSON` out of the body or send `unsupportedMediaType`.
5. We'll get the index of the card in our storage array.
6. If the card is not found, we'll send a `notFound` status.
7. Like in our `PUT`, we'll create a BaseballCard struct out of the `SwiftyJSON` dictionary. Unlike before, we'll reuse the `id` from the url.
8. We'll replace the card at that index with the new card

### `DELETE` /v1/card/:id (3d) (XCODE)

1. We'll create an endpoing for `DELETE` `v1/card/:id`.
2. Check for the `id` in the request URL.
3. Get the index of the card in our storage array.
4. If it is not found, we'll send `notFound`.
5. Remove the card at that index and send `OK` response.

At this point, though, if we stop Xcode and restart, the card at that ID is no longer there.
Because we are storing it in a variable for that runtime, we aren't persisting anything anywhere. This becomes problematic if the server crashes or we lose power.

### Persistence

In order to save our model objects, we'll be using Couch DB.

#### Couch DB

I don't want to get too in-depth here so think of CouchDB as a collection of JSON documents that have:

* a ID
* a revision

##### Installation

1. Go to http://couchdb.apache.org and download the installer.
2. Install and up will pop the dashboard in your browser at 127.0.0.1
3. In the top right, select “Create Database” and name the database “baseball_cards”

##### Kitura (4)

Luckily, Kitura has a middleware plugin for CouchDB already developed which makes this easier.
Unluckily, it can be a little convoluted.

1. We import `CouchDB`
2. We create a `ConnectionProperty` object with the host and port that is in our browser for our CouchDB admin. We are unsecure for now.
3. Instantiate a `CouchDBClient` with those properties.
4. Finally get the database with the name we used in the admin.

### `PUT` /api/v2/card/ (4a) (XCODE)

1. Because we could later have a `completeSet` or `BubbleGum` or `pack` type, we want to have some indication of what the json is in CouchDB. So we set a `type`.
2. Database will create a document from the SwiftyJSON object
3. In the closure, we check an optional revision and document. If they are not there, we send `internalServerError`.
4. If all is good, we send the `id` like before.

### `GET` api/v2/card/:id (4b) (XCODE)

1. Database will retrieve the document with the `id` that was in the URL path.
2. In the closure, we check that there is an document and that it doesn't contain an error.
3. We use the convenience method to transform the CouchDB SwiftyJSON to something that the client is expecting and send it.
4. Otherwise, we send `notFound`.

### `POST` api/v2/card/:id (4c) (XCODE)

1. In order to update, we need to know what the last revision of the document was. So we retrieve using the `id`.
2. We gather the revision from the resulting document
3. We still set the type like we did on creation.
4. Database then updates using the `id` and `revision`
5. Check for document and new revision and that it is not the same as the old revision.
6. If any of that goes awry, we send `internalServerError`.
7. Otherwise, we send OK.

### `DELETE` api/v2/card/:id (4d) (XCODE)

1. Much like `POST`, we need the revision in order to delete so we retrieve using `id`.
2. Database deletes using `id` and `revision`
3. If there is an error, we send `internalServerError`.
4. Otherwise, we send OK

**WATER BREAK**

### Images

Viewing raw JSON is all well and good but what's a card without an image?

#### `PUT` card_image (XCODE) (5a)

1. Just like before, we're going to want to use the body parser to take the request's body and wrap it into an enum.
2. We want to check a few things here:
    * That the path has an `id`
    * That the Content-Type in the header is `image/jpg`
    * That the request has a body and that it is of type raw data
3. We're going to need the latest revision of the CouchDB document that we're trying to access. In order to do that, we retrieve the document using the ID we were given. The call back will give us that revision in the document.
4. Finally, we create what is called an “attachment”. Think of this like an email attachment that we'll be able to pull later.
5. In the callback, we check to make sure no errors are present and, if so, we send an ok status.

#### `GET` card_image (XCODE) (5b)

1. For the get, we want to check that the path has an `id` and the accept type is correct.
2. After that, we do the reverse call where we retrieve the attachment using the name and id.
3. We check the callback for an error. Usually, I'll check if the data is present but what I found is that if no image is found, it gives back a JSON response indicating so.
4. If all is good, I'll send back the image data.  


### Credentials

What if we want a little bit of ownership over our cards? To see which cards are ours and keep private what we own?

In order to do that, we'll put another dependency into the mix.

#### Kitura Credentials (6)

* Framework
* Submodules for each service (Facebook, Google, Twitter, etc…)
* Allows 3rd party to handle authentication, user management, and security
* Mobile Flow
    - We have an endpoint that is behind an authentication check, if the client tries to make a call without the proper credentials, we'll say it's no good.
    - The client will handle authentication through third party and gather a token
    - Client will send a request to an endpoint that is protected but with the token in the request headers
    - Kitura will use Kitura Credentials to check if the token is still valid and cache the results
    - Finally, continue to the endpoint they first called
    - If a bad token is used, we'll ask twitter which will notify us that it is unauthorized and we'll block it

#### But First…

* Go to https://apps.twitter.com and create a project.
* You can use any address as a placeholder.
* You'll need the API keys for your project.

#### Backdoor (XCODE)

* I actually enabled an endpoint called `/faux_login/` that uses the other version of the Twitter Credentials Plugin where it uses a web browser to go to the server which redirects me to Twitter to login and return those keys back to the server which I display on the screen.
* Your client would do this instead.
* It is only for the talk, don't do at home.

#### Implementation (XCODE) (6a)

* Add Kitura-CredentialsTwitter to your Package manifest.
* Add `userID` property to BaseballCard
    - Property and all of the corresponding places it is touched (dictionary, json representation)
* Delete existing documents in DB given `userID` doesn't exist
* Add file called `Keys.swift`
    - Add `Keys.swift` to `.gitignore`
    - Add `let consumerKey` and `let consumerSecret` to `Keys.swift`

#### Credentials Setup (6a)

1. Add `import KituraSession, Credentials, Kitura_CredentialsTwitter`
2. Add a `Session` instance. This session will hold on to user information. We kick it off with a random string.
3. Instantiate a `CredentialsTwitterVerify` instance with our `consumerKey` & `consumerSecret`
4. Create a `Credentials` instance and register our plugin.
5. Add the `credentials` as middleware to the `PUT` `/card` request

#### `PUT` `api/v3/card` Authenticated (XCODE) (6a)

1. The request now has a session and that session has an optional `userProfile` that is filled when it goes to twitter to confirm
2. We'll use the `userProfile`'s `id` to associate with the new card's `userID`

#### `GET` `api/v3/card/:id` Authenticated (XCODE) (6b)

1. Add `credentials` as middleware to `GET` `/card/:id` request
2. Get that `userProfile` like before, you'll need it later
3. You'll retrieve the document like before but check the `userID` on the retrieved document.
4. If it is not the user's document, you'll end an `unauthorized` response.

#### `POST` `api/v3/card/:id` Authenticated (XCODE) (6d)

1. Add `credentials` as middleware to `GET` `/card/:id` request
2. Get that `userProfile` like before, you'll need it later
3. You'll retrieve the document like before but check the `userID` on the retrieved document.
4. If it is not the user's document, you'll end an `unauthorized` response.
5. We want to make extra sure that the `userID` is in the model so we slot it in just in case.

#### `DELETE` `/card:id` Authenticated (XCODE) (6e)
1. Add `credentials` as middleware to `DELETE` `/card/:id` request
2. Get that `userProfile` like before, you'll need it later
3. You'll retrieve the document like before but check the `userID` on the retrieved document.
4. If it is not the user's document, you'll end an `unauthorized` response.

## What Did We Learn?

* Kitura is a Server-Side Swift Application
* All about HTTP
* How to make an API for a basic api webservice
* A little about authentication

## Shoulda/Coulda

* Sending 404, 500, etc…
    * The response technically worked so we’d send an error message
* Very WET code, would consolidate
* Unit Testing!
    * Wrap our service in an object for dependency injection
* Front-End
    * Kitura allows for rendering engines
    * Would prefix api endpoints with `/api/V1/`
