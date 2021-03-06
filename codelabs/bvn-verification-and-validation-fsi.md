summary: Performing BVN Verification and Validation using the NIBSS FSI Sandbox
id: bvn-verification-and-validation-fsi
categories: Web
tags: fsi-sandbox
status: Published
authors: Victor Akinyemi
feedback link: https://dscfuta.com

# Performing BVN Verification and Validation using the NIBSS FSI Sandbox

## Overview

Duration: 1

In this codelab, we shall be taking a look at how to perform actions such as BVN verification and validation using the FSI Sandbox API by constructing a wrapper API around it.

A live example of what we will build is available at: `https://dsc-fsi-wrapper.herokuapp.com/api`

Negative
: **N/B:** The sandbox API is restricted and only provides a simulation of what the actual API works like. Therefore, only data present in the sandbox can be used, this includes BVNs and user data.

### Resources

- [Documentation for the Wrapper API](https://documenter.getpostman.com/view/9936833/SWLce9RJ?version=latest)
- [FSI Sandbox](https://sandbox.fsi.ng)
- [Source code for the Wrapper API](https://github.com/dscfuta/dsc-fsi-wrapper)

## Project Setup

Duration: 5

For the purposes of this codelab, we will be using NodeJS and Express.js.

Create a folder for this project and cd into it:

```
$ mkdir fsi-codelab && cd fsi-codelab
```

Then initialize an npm project in the directory. We're accepting all default options here since they don't really matter.
We also install some dependencies, including `nodemon` to automatically restart the project when we make changes to our code, and `node-fetch`, an implementation of the `fetch` API for Node, and `dot-env` for reading environment variables from `.env` files.

```
$ npm init -y && npm install --save express node-fetch dot-env && npm install --save-dev nodemon
```

Create a file called `index.js` and type (or copy-paste) the following inside:

```javascript
require("dot-env").config();
const app = require("express")();

app.get("/", function(req, res) {
  res.send("Hello World!");
});

app.listen(3000, function() {
  console.log(`App listening on *:3000`);
});
```

Then create a file named `.env` with the following content:
```
API_URL=https://sandboxapi.fsi.ng/nibss
ORGANISATION_CODE=11111
SANDBOX_KEY=[your FSI Sandbox key which can be found on your profile]
```

Positive
: **N/B**: As a good rule of thumb, make sure to add this file to your .gitignore. Environment variables should *never* be checked in to a Git repo.

Then run `npx nodemon`, which will start our program and automatically restart it when we make any changes.

Positive
: **N/B**: `npx` is a command that comes with `npm` that allows us to run binaries in the `node_modules` folder without adding `./node_modules/.bin` to our path. The equivalent yarn command would simply be `yarn run nodemon`. `npx` also allows you to run a binary from a non-installed package. You can read more [here](https://medium.com/@maybekatz/introducing-npx-an-npm-package-runner-55f7d4bd282b).

## How to communicate with the sandbox API

Duration: 10

### Introduction

In this section, you will learn how to communicate with the sandbox API. The API is mostly a normal RESTful API, with some additional requirements for security. Some additional headers are required, and requests are encrypted before sending them and decrypted once received. The keys for performing these operations are generated by the server in a process that is outlined below.

### Communicating with the API - Generating encryption keys

All categories of requests on the API (/bvnr for BVN verification, /BVNPlaceHolder for BVN matching, /fp for Fingerprint validation respectively) have an associated `/Reset` route that doesn't require an encrypted request body.

This `/Reset` route is used to generate the encryption keys which are then used henceforth for all requests to any other route in that category. The process is as follows:

1. Make a request to /Reset with the following headers:
  - `OrganisationCode`: Your organisation code (`11111` in the case of the FSI Sandbox), in base64 encoded format.
  - `Sandbox-Key`: Your sandbox key, which can be gotten from your profile on the [FSI Sandbox](http://sandbox.fsi.ng).
2. If the details are correct, a response with status 200 OK is returned with the following extra headers
  - `name`: The name of the organisation
  - `ivkey`: One of the pair of keys used in the encryption process
  - `aes_key`: One of the pair of keys used in the encryption process
  - `password`: This password will be used when generating some of the required headers
  - `email`: The email associated with the organisation
  - `code`: The organisation code
3. We generate the required headers using the above details:
  - `OrganisationCode`: Same as before.
  - `Sandbox-Key`: Same as before.
  - `SIGNATURE`: This is a SHA256 hashed string that consists of the organisation code, concatenated with today's date in YYYYMMDD format, then concatenated with the password. Basically: `const SIGNATURE = hash(code + "20200101" + password);`
  - `SIGNATURE_METH`: The method with which the `SIGNATURE` header was generated, `SHA256` is the only supported method at the moment.
  - `Authorization`: A base64 encoded string consisting of the `code` and `password` separated by a colon. That is, `${code}:${password}`.
  - `Content-Type`: This will be `application/json`, since we are sending JSON. The API also supports requests in XML format, but that is out of the scope of this codelab.
  - `Accept`: This will also be `application/json` as we require a JSON response from the server.
  - We create a cipher and decipher using the AES key and IV key provided to us, and use those to decrypt and encrypt futher requests.


### Code sample

```javascript
// Store this as performReset.js
const crypto = require("crypto");

// Converts a given string to the base64 format
const toBase64 = string => Buffer.from(string).toString("base64");

// Hashes a given string with SHA256
const sha256 = string => {
  const hash = crypto.createHash("sha256");
  hash.update(string);
  return hash.digest("hex");
};

// Takes an AES key and IV key and returns a function that
// encrypts anything it's given using said keys.
const aesEncrypt = (aesKey, ivKey) => text => {
  const cipher = crypto.createCipheriv(
    "aes-128-cbc",
    Buffer.from(aesKey),
    ivKey
  );
  let encrypted = cipher.update(text);
  encrypted = Buffer.concat([encrypted, cipher.final()]);
  return encrypted.toString("hex");
};

// Takes an AES key and IV key and returns a function that
// decrypts anything it's given using said keys.
const aesDecrypt = (aesKey, ivKey) => text => {
  const encryptedText = Buffer.from(text, "hex");
  let decipher = crypto.createDecipheriv(
    "aes-128-cbc",
    Buffer.from(aesKey),
    ivKey
  );
  let decrypted = decipher.update(encryptedText);
  decrypted = Buffer.concat([decrypted, decipher.final()]);
  return decrypted.toString();
};

// Returns the date formatted in a YYYYMMDD mode
const getTodaysDate = () =>
  new Date()
    .toJSON()
    .slice(0, 10)
    .replace(/-/g, "");

// Pass these in as environment variables in your .env file
const API_URL = process.env.NODE_ENV;
const organisationCode = process.env.ORGANISATION_CODE;
const sandboxKey = process.env.sandboxKey;

// type is one of bvnr, BVNPlaceHolder, or fp
// as defined in the documentation
export function performReset(type = "bvnr") {
  return new Promise((resolve, reject) => {
    fetch(`${API_URL}/${type}/Reset`, {
      method: "POST",
      headers: {
        // Make a request to generate our encryption keys
        OrganisationCode: toBase64(organisationCode),
        "Sandbox-Key": sandboxKey
      }
    })
      .then(res => {
        // If we have a successful request
        if (res.status === 200) {
          const header = name => res.headers.get(name);
          const details = {
            name: header("name"),
            ivKey: header("ivkey"),
            aesKey: header("aes_key"),
            password: header("password"),
            email: header("email"),
            code: header("code"),
            name: header("name")
          };
          // Use the data returned to generate our headers
          const getRequestHeaders = () => {
            const headers = {
              OrganisationCode: toBase64(organisationCode),
              "Sandbox-Key": sandboxKey,
              SIGNATURE: sha256(
                details.code + getTodaysDate() + details.password
              ),
              SIGNATURE_METH: "SHA256",
              Authorization: toBase64(`${details.code}:${details.password}`),
              "Content-Type": "application/json",
              Accept: "application/json"
            };
            return headers;
          };
          // Create functions for encrypting and decrypting using those details
          const encryptBody = aesEncrypt(details.aesKey, details.ivKey);
          const decryptBody = aesDecrypt(details.aesKey, details.ivKey);
          const credentials = {
            details,
            getRequestHeaders,
            encryptBody,
            decryptBody
          };
          resolve(credentials);
        } else {
          res.text().then(res => {
            reject(res);
          });
        }
      })
      .catch(err => {
        reject(err);
      });
  });
}
```

This reset must be called once for every category before any other request is made. One way to ensure this is to make a request to all the reset endpoints and store the results to be used across the app before calling `app.listen`, or to use a middleware to call the endpoint if required and inject the results in to the `Request` object. We will be taking the latter approach in this codelab.


### Pre-request preparations

As discussed in the last section, we will be using a middleware to perform a reset if required before all our requests, store the result, and inject it into the `Request` object.

Create a file called resetMiddleware.js with the following contents:
```javascript
const { performReset } = require('./performReset.js');

// We'll store any already peformed resets in here to check for later
const _resetCache = {
  bvnr: null,
  BVNPlaceHolder: null,
  fp: null
}

// This function creates a middleware that performs a reset of a particular type
// when required
export function resetIfNeeded(type = "bvnr") {
  return function reset(req, res, next) {
    // Have we performed a reset already for this endpoint family?
    // If so, used the cached values
    if (_resetCache[type] != null) {
      req.apiCredentials = _resetCache[type];
      next();
    }
    // Otherwise perform a reset request and cache that
    else {
      performReset(type).then(cred => {
        _resetCache[type] = cred;
        req.apiCredentials = _resetCache[type];
        next();
      }).catch(next);
    }
  }
}
```

Now we can simply define a route like so, and access `req.apiCredentials` in the route handler:

```javascript
app.get('/verifyBVN', resetIfNeeded("bvnr"), ...);
```

## Performing a request
Duration: 10

### Overview

We will be using everything from the previous requests to make a request to the sandbox API in this step. We shall be verifying a single BVN using the /bvnr/VerifySingleBVN endpoint.

### Code

```javascript
// Near the top
const { resetIfNeeded } = require('./resetIfNeeded');
const fetch = require('node-fetch');
const API_URL = process.env.API_URL;

// ...


app.post('/verify/:BVN', resetIfNeeded("bvnr"), function (req, res) {
  const { encryptBody, decryptBody, getRequestHeaders } = req.apiCredentials;
  const BVN = req.params.BVN;
  // The API expects a JSON object with a BVN key containing the BVN to verify
  const body = JSON.stringify({
    BVN
  })
  fetch(`${API_URL}/bvnr/VerifySingleBVN`, {
    method: 'POST',
    headers: {
      ...getRequestHeaders()
    },
    body: encryptBody(body)
  })
    .then(res => {
      // Normally, you'd be calling res.json() here
      // But the sandbox returns an encrypted response that we need to decrypt
      // So we use the text() function to get the response as a string of text
      res.text().then(encryptedRes => {
        // Decrypt the response...
        const resp = decryptBody(encryptedRes);
        // ...and convert it to JSON
        const jsonRes = JSON.parse(resp);
        console.log(jsonRes);
      })
    })
    // Pass on any errors to Express' error handler
    .catch(next);
})
```

You can test this by making a POST request to `http://localhost:3000/verify/12345678901` with your favourite API testing tool (Postman and Insomnia, for example), and checking your console after. The response should be of this format, if all goes well:

```javascript
{
  message: 'OK',
  data: {
    ResponseCode: '00',
    BVN: '12345678901',
    FirstName: 'Uchenna',
    MiddleName: 'Chijioke',
    LastName: 'Nwanyanwu',
    DateOfBirth: '22-Oct-1970',
    PhoneNumber: '07033333333',
    RegistrationDate: '16-Nov-2014',
    EnrollmentBank: '900',
    EnrollmentBranch: 'Victoria Island',
    WatchListed: 'NO'
  }
}
```

Note that, as specified earlier, the sandbox contains only sample data, as such, if you try to verify your own BVN for example, you would get a response like this:

```javascript
{
  ResponseCode: '05',
  Message: 'Unmatched Request, Refer to documentation.',
  EXPECT: {
    header: {
      Accept: ['application/xml', 'application/json'],
      'Content-Type': ['application/xml', 'application/json'],
      OrganisationCode: '...',
      Authorization: '...',
      SIGNATURE: '...',
      SIGNATURE_METH: 'SHA256'
    },
    body: { BVN: '12345678901' }
  }
}
```

What this means is essentially that you are only allowed to make a request to verify the sample BVN `12345678901`.

### Response format

Let's look at the format of the response we got from the API in the previous section.

```javascript
{
  message: 'OK',
  data: {
    ResponseCode: '00',
    BVN: '12345678901',
    FirstName: 'Uchenna',
    MiddleName: 'Chijioke',
    LastName: 'Nwanyanwu',
    DateOfBirth: '22-Oct-1970',
    PhoneNumber: '07033333333',
    RegistrationDate: '16-Nov-2014',
    EnrollmentBank: '900',
    EnrollmentBranch: 'Victoria Island',
    WatchListed: 'NO'
  }
}
```

- `Message`: This is `OK` for a successful response, or a descriptive error message if an error occured.
- `ResponseCode`: Always `00` for successful requests, other wise it will be a two-digit error code. The list of error codes can be found on the NIBSS API documentation available for download on the FSI Sandbox.
- `EXPECT`: Sandbox-specific. If the request you sent doesn't match what the sandbox expects, this field is present and it contains the headers and request body the sandbox API expects you to send.
- `data`: This contains the response data for the request. Excluding `ResponseCode`, the structure of this object is request specific. Requests that handle multiple BVNs return an array of objects instead of an object. Note that each object will have it's own `ResponseCode`.

You can then consume the data from the API and use them however you want in your application. You may not even expose the endpoint and just have it as a function in your backend code that's called as part of some other process in your app. The posibilites are limitless (but remember, the sandbox works only with sample data).

## Conclusion
Duration: 1

The process is similar for the other endpoints, as outline below:
- Make a request to `/Reset` at one point in the lifecycle of the app, before making any further requests, for each group of endpoints
  We handle this by making the request when needed and injecting the response into to the `Request` object using a middleware
- Perform requests to the API, making sure to encrypt and decrypt the response using the data from the `/Reset` response.
  Functions to do this are also injected into the `Request` object by our middleware.

And that's it. Have fun building your apps! :)