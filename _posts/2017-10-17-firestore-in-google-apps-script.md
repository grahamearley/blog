---
layout: post
title:  "Accessing Firestore in a Google Apps Script"
date:   2017-10-18 00:13:37 -0600
---

In previous posts ([here](http://grahamearley.website/blog/2016/08/17/firebase-notifications-google-docs.html) and [here](http://grahamearley.website/blog/2016/07/17/firebase-cms-interface.html)), I have discussed my Google Sheets solution for interfacing with Firebase's Realtime Database. I am working on an [Android app](https://play.google.com/store/apps/details?id=org.jnanaprabodhini.happyteacherapp) that relies on some content managers, and the familiar spreadsheet interface of Google Sheets provides them with an easy way to edit content that will appear in our app, and the added structure of the spreadsheet along with a custom Google Apps Script gives me peace of mind that everything is being properly formatted in the database.

When [Firestore](https://firebase.google.com/docs/firestore/) was announced, I wanted to migrate to it ASAP (being able to use multiple "where" clauses in a query would be so nice!). However, in order to do so, I need to re-work my spreadsheet so that it writes to Firestore instead of the Firebase Realtime Database (which I'm going to call RDB from now on).

My spreadsheet relies on an unofficial [Google Apps Script library](https://sites.google.com/site/scriptsexamples/new-connectors-to-google-services/firebase) for interfacing with the Firebase RDB. Since Firestore is so new, it appears that no such library exists for it yet. So I decided to [start writing one](https://github.com/grahamearley/FirestoreGoogleAppsScript).

In this post, I'm going to go over some details for authenticating a service account (so that my spreadsheet can be authorized to make edits) and writing to Firestore using the Firestore REST API in a Google Apps Script.

<!--more-->

## Authenticating a service account and obtaining a token
For this step, I'm going to assume your Firestore [security rules](https://firebase.google.com/docs/firestore/security/get-started) require authentication for reads and writes. If your database can be read or written to by anyone, then you don't have to worry about this step (but you might have to worry about the security of your data!).

When making a call to the Firestore REST API, you will need to provide an `Authorization: Bearer` HTTP header with a token for the user accessing the data. Since in this case we're using a Google Apps Script to access the data, we will use a "service account" to represent this script as a user. We need this service account to generate our token.

**Note:** _This section is largely going to be following along [this documentation](https://developers.google.com/identity/protocols/OAuth2ServiceAccount), but with the specific implementation details for doing so for Firestore and in a Google Apps Script._

The ["Creating a service account"](https://developers.google.com/identity/protocols/OAuth2ServiceAccount#creatinganaccount) section of [this page](https://developers.google.com/identity/protocols/OAuth2ServiceAccount) gives good instructions on how to create a service account. When you're following these instructions to make a service account, ensure that you give the account full access to the `https://www.googleapis.com/auth/datastore` scope. To do this, select the `Datastore > Cloud Datastore Owner` role for your service account (in the "Create service account" window of the [service accounts page](https://console.developers.google.com/permissions/serviceaccounts)).

When the account is created, you will get a JSON file with your account's information -- most importantly, the key (under `private_key`) and the service account email address (under `client_email`). Keep these! We will be using them later.

To obtain a token from this service account, we follow along on [this section](https://developers.google.com/identity/protocols/OAuth2ServiceAccount#authorizingrequests) following the ["Creating a service account"](https://developers.google.com/identity/protocols/OAuth2ServiceAccount#creatinganaccount) section.

As the documentation says, there are three steps to get our token:

> 1. Create a JSON Web Token (JWT, pronounced "jot"), which includes a header, a claim set, and a signature.
> 2. Request an access token from the Google OAuth 2.0 Authorization Server.
> 3. Handle the JSON response that the Authorization Server returns.

### Creating the JWT
To create the JWT in our Google Apps Script, we'll write a function `createJwt(email, key)` that takes the service account email and key as parameters. The JWT consists of three things: a header, a claim set, and a signature, all web-safe Base64 encoded and joined together by periods.

The header is easy because it is always the same (read more in the documentation if you want to know more!). For service accounts, it will always be `{"alg":"RS256", "typ":"JWT"}`. So, we'll add a line to our function to store this header:

{% highlight javascript %}
const jwtHeader = {"alg" : "RS256", "typ" : "JWT"};
{% endhighlight %}

For the JWT claim, we need the email that was passed into this function. We also need to set the date and time the claim was made, along with the date and time that the claim expires. We'll set the claim to be made now and to expire in one hour.  The claim requires these `Date`s to be formatted as seconds from that [fateful New Year's Day in 1970](https://en.wikipedia.org/wiki/Year_2038_problem). Let's create those `Date` values before making the object.

{% highlight javascript %}
const now = new Date();

// Divide by 1000 to convert from milliseconds to seconds!
const nowSeconds = now.getTime() / 1000;
 
now.setHours(now.getHours() + 1);
const oneHourFromNowSeconds = now.getTime() / 1000;
{% endhighlight  %}

Now, we can create our claim! It is an object that looks like this:

{% highlight javascript %}
const jwtClaim = {
    "iss" : email, // the email parameter that was passed in
    "scope" : "https://www.googleapis.com/auth/datastore",
    "aud" : "https://www.googleapis.com/oauth2/v4/token/",
    "exp" : oneHourFromNowSeconds,
    "iat" : nowSeconds
  }
{% endhighlight  %}

Continuing to follow the documentation's instructions, we need to Base64 encode the stringified header and claim, and then join them with a period. This will form the input string for our signature, which we will sign using the private key passed in as a parameter.

However, as I ([and others](https://stackoverflow.com/a/37604977/5054197)) discovered, you have to be careful here of how you Base64 encode things. There are two different ways to do it using the Google Apps Script [`Utilities` class](https://developers.google.com/apps-script/reference/utilities/). We want to format this as web-safe, so we will use [`Utilities.base64EncodeWebSafe(data)`](https://developers.google.com/apps-script/reference/utilities/utilities#base64encodewebsafedata). This method still includes `=` symbols in the encoded output, though. Our token should not have these. To handle the removal of `=`s from our output, I wrote a helper function: 

{% highlight javascript %}
function base64EncodeSafe(string) {
  var encoded = Utilities.base64EncodeWebSafe(string);
  return encoded.replace(/=/g, "");
}
{% endhighlight  %}

Phew! Now that we have this function, we can use it in our `createJwt(email, key)` function to encode our data and join it with a `.` in between:

{% highlight javascript %}
const jwtHeaderBase64 = base64EncodeSafe(JSON.stringify(jwtHeader));
const jwtClaimBase64 = base64EncodeSafe(JSON.stringify(jwtClaim));

const signatureInput = jwtHeaderBase64 + "." + jwtClaimBase64;
{% endhighlight  %}

This `signatureInput` value needs to be signed by RSA using SHA-256 hashing algorithm, with our passed-in key as the key. Luckily, there's a `Utilities` function to do that for us. Let's use it:

{% highlight javascript %}
const signature = Utilities.computeRsaSha256Signature(signatureInput, key);
{% endhighlight  %}

Now, all we need to do is encode this signature, and join it with our other `.`-joined string, `signatureInput`, to create our final JWT, structured as  `{Base64url encoded header}.{Base64url encoded claim set}.{Base64url encoded signature}`.

The code for that looks like this:

{% highlight javascript %}
const encodedSignature = base64EncodeSafe(signature);
const jwt = signatureInput + "." + encodedSignature;
return jwt;
{% endhighlight  %}

Putting this all together, the full function looks like this:

{% highlight javascript %}
function createJwt(email, key) {
  const jwtHeader = {"alg" : "RS256", "typ" : "JWT"};
  
  const now = new Date();
  const nowSeconds = now.getTime() / 1000;
  
  now.setHours(now.getHours() + 1);
  const oneHourFromNowSeconds = now.getTime() / 1000;
  
  const jwtClaim = {
    "iss" : email,
    "scope" : "https://www.googleapis.com/auth/datastore",
    "aud" : "https://www.googleapis.com/oauth2/v4/token/",
    "exp" : oneHourFromNowSeconds,
    "iat" : nowSeconds
  }
  
  const jwtHeaderBase64 = base64EncodeSafe(JSON.stringify(jwtHeader));  
  const jwtClaimBase64 = base64EncodeSafe(JSON.stringify(jwtClaim));
  
  const signatureInput = jwtHeaderBase64 + "." + jwtClaimBase64;
  
  const signature = Utilities.computeRsaSha256Signature(signatureInput, key);
  const encodedSignature = base64EncodeSafe(signature);
  
  const jwt = signatureInput + "." + encodedSignature;
        
  return jwt;
}
{% endhighlight  %}

### Requesting and receiving the auth token
With this JWT, we can now get the auth token. So let's write a function called `getAuthToken(email, key)` (we'll use `email` and `key` as parameters because we'll need them to call `createJwt(email, key)` within this function).

This function has just one important piece: the HTTP request options. After obtaining our JWT and storing it to `jwt`, our payload looks like this:

{% highlight javascript %}
const options = {
   'method' : 'post',
   'payload' : 'grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-bearer&assertion=' + encodedJwt,
   'muteHttpExceptions' : true
};
{% endhighlight  %}

The first entry in the object sets the `method` to POST. The last entry mutes HTTP exceptions, so that our Google Apps Script gives more detailed errors if anything goes wrong in the request.

The *middle* entry, the payload, is what's important here. The first part of the payload, `'grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-bearer&assertion='` is constant (see [here](https://developers.google.com/identity/protocols/OAuth2ServiceAccount#makingrequest)). After this part, we simply tack on our JWT as our `assertion`. Now the request is ready to send! Adding in the code for sending this request and receiving a response (and returning the access token from the response, assuming there is no error), our function looks like this:

{% highlight javascript %}
function getAuthToken(email, key) {
  const jwt = createJwt(email, key);
  
  const options = {
   'method' : 'post',
   'payload' : 'grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-bearer&assertion=' + jwt,
   'muteHttpExceptions' : true
  };
  
  const response = UrlFetchApp.fetch("https://www.googleapis.com/oauth2/v4/token/", options)
  const responseObj = JSON.parse(response.getContentText())
  
  return responseObj["access_token"];
}
{% endhighlight  %}

Now we've got a function for generating auth tokens for our service account! Next, we'll put it into action by using this token to write to Firestore.


## Creating a Firestore document using the REST API in a Google Apps Script
You can find the official reference for the Firestore REST API [here](https://firebase.google.com/docs/firestore/use-rest-api). However, the API works differently than the Firebase RDB REST API, and the documentation does not yet seem to have examples. This section will serve as one such example. We'll be using the API, combined with our above `getAuthToken(email, key)` function, to [create a new Firestore document](https://firebase.google.com/docs/reference/rest/firestore/v1beta1/projects.databases.documents/createDocument). We'll put this logic in a `createDocument(path, documentId, documentData, email, key)` function (the `email` and `key` are being passed in again to get the auth token).

As shown in the [create document](https://firebase.google.com/docs/reference/rest/firestore/v1beta1/projects.databases.documents/createDocument) reference, there is one required parameter for this call: `documentId`. We'll assume that our parent resource is the root collection in the database, with URL `https://firestore.googleapis.com/v1beta1/projects/{YOUR PROJECT ID HERE}/databases/(default)/documents/`. From this, we can tack on any additional path to a document. Finally, we'll add on the `documentId` parameter. Thus, the we're sending our request to:

{% highlight javascript %}
const baseUrl = "https://firestore.googleapis.com/v1beta1/projects/{YOUR PROJECT ID HERE}/databases/(default)/documents/" + path + "?documentId=" + documentId;
{% endhighlight  %}

This function ought to be pretty simple now that we have this URL. The data for the document was already passed in! All we need to do is make a POST request with an `Authorization: Bearer` HTTP header.

Well, there's actually one more step. Let's talk about the `documentData` we're passing in here.

### Formatting document data for Firestore
In our `createDocument` function, the `documentData` parameter is assumed to be a JSON object similar to what you would write to a Firebase reference. Keys map to field names, and values map to their values. For example, our data could be

{% highlight javascript %}
const documentData = {
   "name" : "Radia Perlman",
   "birthYear" : 1951,
   "isCool" : true
};
{% endhighlight  %}

This is a nice and simple format for setting the fields of our document, so that's how we're going to pass the data into our function. However, this is *not* how Firestore wants our data to be formatted! We have to do some extra work in our function. In fact, we're going to create a few other functions to help with formatting our field data so that it works with Firestore. We'll come back to finish the `createDocument` function afterward.

#### Converting from a key-value map to Firestore document fields
The [Document](https://firebase.google.com/docs/reference/rest/firestore/v1beta1/Document) page of the REST API documentation is not very clear about how to format Documents, in my opinion. Using the [API Explorer](https://developers.google.com/apis-explorer/#search/firestore/firestore/v1beta1/) was more illuminating. With this tool, I learned that Firestore wants a Document's `fields` field to be a slightly more complicated version of the simple key-value map that we're passing into the `createDocument` function we started writing. This slightly more complicated format wraps each value in an object with a single key denoting the value's type. The value of this key is the original value.

So, if we want our document to have a string field `"name"` with value `"Radia Perlman"`, then our Firestore-ready object should look like this:

{% highlight javascript %}
const firestoreReadyDocument = {
  "fields" : {
     "name" : {
       "stringValue" : "Radia Perlman"
     }
  }
};
{% endhighlight  %}

All we changed from the `documentData` object above was to make `"name"` point to an object `{"stringValue": "Radia Perlman"}` instead of the plain string `"Radia Perlman"`, and then to put all this as a value for the key `"fields"` (you can put other information in a Document, such as an edit timestamp, but we will only look at fields in this post). 

Let's make some helper functions for wrapping basic values according to how Firestore wants them.

{% highlight javascript %}
function wrapString(string) {
  return {"stringValue" : string};
}

function wrapBoolean(boolean) {
  return {"booleanValue" : boolean};
}

function wrapInt(int) {
  return {"integerValue" : int};
}

function wrapDouble(double) {
  return {"doubleValue" : double};
}

function wrapNumber(num) {
  if (isInt(num)) {
    return wrapInt(num);
  } else {
    return wrapDouble(num);           
  }
}

function wrapObject(object) {
  if (!object) {
    return {"nullValue" : null};
  }
  
  // `createFirestoreObject(object)` is calling a function we will write next. Read on!
  return {"mapValue" : createFirestoreObject(object)};
}
{% endhighlight  %}

In this last function, `wrapObject`, I call `createFirestoreObject(object)`, which is a function we haven't written yet (but we will next!). This is the function that will create a Firestore-ready Document, and we call it here because if you set a field to be a JSON object, Firestore expects the value of that field to be formatted in the same way as the base document.

In the above code, I also use a helper function for determining if a number is a double or an integer. This function is:

{% highlight javascript %}
// Assumes n is a Number.
function isInt_(n) {
   return n % 1 === 0;
}
{% endhighlight %}

Okay! With these functions, let's define our big function `createFirestoreObject(object)`, which will take a typical key-value field map object and format it to be a Firestore object.

This function takes an object `object` as a parameter. First, it creates the object `firestoreObj` that we will populate and return. Then, for each key `key` in `object,` we wrap the value at `key` according to its type and put that wrapped object as the value at `firestoreObj["fields"][key]`. All together, it looks like this:

{% highlight javascript %}
function createFirestoreObject(object) {
  const keys = Object.keys(object);
  const firestoreObj = {};
    
  firestoreObj["fields"] = {};
    
  for (var i = 0; i < keys.length; i++) {
    var key = keys[i];
    var val = object[key];
    
    var type = typeof(val);
    
    switch(type) {
      case "string":
        firestoreObj["fields"][key] = wrapString(val);
        break;
      case "object":
        firestoreObj["fields"][key] = wrapObject(val);
        break;
      case "number":
        firestoreObj["fields"][key] = wrapNumber(val);
        break;
      case "boolean":
        firestoreObj["fields"][key] = wrapBoolean(val);
        break;
      default:
        break;
    }
  }
    
  return firestoreObj
}
{% endhighlight %}

**Note:** _This function only handles fields that are strings, numbers, booleans, null, or objects with fields consisting of these types. But other types can be easily added -- and you can [contribute here!](https://github.com/grahamearley/FirestoreGoogleAppsScript)_

### Finishing the document creation function
After that necessary digression about how to format fields for Firestore, we can finish that `createDocument` function we started! Recall that, aside from formatting the fields correctly, all we needed to do was make a POST request with an `Authorization: Bearer` HTTP header.

It's pretty simple to do that. After getting our auth token and formatting our `documentData` using the functions we wrote above, we can create our request options:

{% highlight javascript %}
const token = getAuthToken(email, key);
const firestoreObject = createFirestoreObject(documentData);
const options = {
   'method' : 'post',
   'muteHttpExceptions' : true,
   'payload': JSON.stringify(firestoreObject),
   'headers': {'content-type': 'application/json', 'Authorization': 'Bearer ' + token}
};
{% endhighlight %}

Now we're ready to send this request off! Combining all this, our `createDocument` function is as follows:

{% highlight javascript %}
function createDocument(path, documentId, documentData, email, key) {
  const token = getAuthToken(email, key);
  
  const firestoreObject = createFirestoreObject(documentData);
  
  const baseUrl = "https://firestore.googleapis.com/v1beta1/projects/{YOUR PROJECT ID HERE}/databases/(default)/documents/" + path + "?documentId=" + documentId;
  const options = {
   'method' : 'post',
   'muteHttpExceptions' : true,
   'payload': JSON.stringify(firestoreObject),
   'headers': {'content-type': 'application/json', 'Authorization': 'Bearer ' + token}
  };
  
  return UrlFetchApp.fetch(baseUrl, options);
}
{% endhighlight %}

## Wrapping up
With these functions written, we have the start to a [Google Apps Script library for interacting with Firestore](https://github.com/grahamearley/FirestoreGoogleAppsScript).

Let's do one final example showing what it looks like to write a document to Firestore.

{% highlight javascript %}
const email = " {YOUR SERVICE ACCOUNT EMAIL HERE} ";
const key = " {YOUR PRIVATE KEY HERE} ";

const documentData = {
   "name" : "Radia Perlman",
   "birthYear" : 1951,
   "isCool" : true
};

createDocument("FirstCollection", "RadiaPerlmanDocument", documentData, email, key);
{% endhighlight %}

The response from this request to create a document in our Firestore database shows the resulting document:

{% highlight javascript %}
document = {
  "name" : "projects/{PROJECT-NAME}/databases/(default)/documents/FirstCollection/RadiaPerlmanDocument",
  "fields" : {
    "isCool" : {
      "booleanValue" : true
    },
    "name" : {
      "stringValue" : "Radia Perlman"
    },
    "birthYear" : {
      "integerValue" : "1951"
    }
  },
  "createTime" : "2017-10-19T04:39:00.600862Z",
  "updateTime" : "2017-10-19T04:39:00.600862Z"
}
{% endhighlight %}

And that's that! We've now got a foot in the door for interfacing with Firestore through a Google Apps Script!

✌ 

Questions? [Email me.](mailto:graham@grahamearley.website)