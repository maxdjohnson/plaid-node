plaid-node
==============

![alt text](https://circleci.com/gh/plaid/plaid-node.png?circle-token=2efcf082d8df7e119325a4dbed9a1091ff5db422)

plaid-node is a node.js client for the [Plaid API](https://plaid.com).

This module was recently refactored and released as version `1.0.0`.  The previous
version, `0.1.1`, is still available via npm.

## Install and setup

```
npm install plaid
```
then (from `/node_modules/plaid`):

```
npm install
```

## Getting started

The module supports all Plaid API endpoints.  For complete information about the API, head
to the [docs](https://plaid.com/docs).

### Public Routes

Public routes (category and institution information) require no authentication and can be accessed as follows:

```javascript
var plaid = require('plaid');

plaid.getCategory(category_id, plaid_env, callback);
plaid.getCategories(plaid_env, callback);

plaid.getInstitution(institution_id, plaid_env, callback);
plaid.getInstitutions(plaid_env, callback);
```

`plaid_env` dictates which Plaid API environment you will access.  Values are:
- `plaid.environments.tartan` - use for integration development and testing
- `plaid.environments.production` - production use, creates users on https://api.plaid.com

Environments are exported from the module, i.e.:

```javascript
var plaid = require('plaid');

console.log(plaid.environments);
```

### Authenticated Routes

Authenticated routes require a valid `client_id` and `secret` to access.  You can use the sandbox
client_id and secret for testing (`test_id` and `test_secret`).

All authenticated routes are accessible from an instance of a Plaid `Client`:

```javascript
var plaid = require('plaid');

var plaidClient = new plaid.Client(client_id, secret, plaid_env);
```

The `plaid_env` parameter dictates which Plaid API environment you will access. Values are:
- `plaid.environments.tartan` - use for integration development and testing, creates users on https://tartan.plaid.com
- `plaid.environments.production` - production use, creates users on https://api.plaid.com

Once an instance of the client has been created you use the following methods:

```javascript
var plaid = require('plaid');

// Initialize client
var plaidClient = new plaid.Client(client_id, secret, plaid_env);

// addAuthUser(String, Object, Object?, Function)
plaidClient.addAuthUser(institution_type, credentials, options, callback);
// stepAuthUser(String, String, Object?, Function)
plaidClient.stepAuthUser(access_token, mfaResponse, options, callback);
// getAuthUser(String, Object?, Function)
plaidClient.getAuthUser(access_token, options, callback);
// patchAuthUser(String, Object, Object? Function)
plaidClient.patchAuthUser(access_token, credentials, options, callback);
// deleteAuthUser(String, Object?, Function)
plaidClient.deleteAuthUser(access_token, options, callback);

// addConnectUser(String, Object, Object?, Function)
plaidClient.addConnectUser(institution_type, credentials, options, callback);
// stepConnectUser(String, String, Object?, Function)
plaidClient.stepConnectUser(access_token, mfaResponse, options, callback);
// getConnectUser(String, Object?, Function)
plaidClient.getConnectUser(access_token, options, callback);
// patchConnectUser(String, Object, Object?, Function)
plaidClient.patchConnectUser(access_token, credentials, options, callback);
// deleteConnectUser(String, Object?, Function)
plaidClient.deleteConnectUser(access_token, options, callback);

// addInfoUser(String, Object, Object?, Function)
plaidClient.addInfoUser(institution_type, credentials, options, callback);
// stepInfoUser(String, String, Object, Function)
plaidClient.stepInfoUser(access_token, mfaResponse, options, callback);
// getInfoUser(String, Object?, Function)
plaidClient.getInfoUser(access_token, options, callback);
// patchInfoUser(String, Object, Object?, Function)
plaidClient.patchInfoUser(access_token, credentials, options, callback);
// deleteInfoUser(String, Object?, Function)
plaidClient.deleteInfoUser(access_token, options, callback);

// getBalance(String, Function)
plaidClient.getBalance(access_token, callback);

// upgradeUser(String, String, Object?, Function)
plaidClient.upgradeUser(access_token, upgrade_to, options, callback);

// exchangeToken(String, Function)
plaidClient.exchangeToken(public_token, callback);
```

**All parameters except `options` are required.**

## Callbacks

For a request that could potentially return a MFA response, callbacks are in the form:

```javascript
function callback(err, mfaResponse, response) {
  // err can be a network error or a Plaid API error (i.e. invalid credentials)
  // mfaResponse can be any type of Plaid MFA flow
}
```
All `add`, `step`, and `patch` related requests can return a MFA response.  `upgradeUser` can also return MFA responses.

For `delete`, `get`, `getBalance`, and `exchangeToken` requests, callbacks are in the form:

```javascript
function callback(err, response) {
  // err can be a network error or a Plaid API error (i.e. invalid credentials)
}
```

## A note on error handling

The `err` argument passed to either callback style can either be an instance of `Error`
or a [Plaid API error](https://plaid.com/docs/#response-codes) object.  An `Error` object
is only passed back in the case of a HTTP connection error.  The following code distinguishes
between a Plaid error and a standard Error instance:

```javascript
function callback(err, response) {
  if (err != null) {
    if (err.code != null) {
      // This is a Plaid error
      console.log(err.code + ': ' + err.message);
    } else {
      // This is a connection error, an Error object
      console.log(err.toString());
    }
  }
}
```

## Examples

Bank of America question-based MFA flow:
```javascript
var plaid = require('plaid');

// Initialize a client
var plaidClient = new plaid.Client('test_id', 'test_secret', plaid.environments.tartan);

// Add a BofA auth user going through question-based MFA
plaidClient.addAuthUser('bofa', {
  username: 'plaid_test',
  password: 'plaid_good',
}, function(err, mfaResponse, response) {
  if (err != null) {
    // Bad request - invalid credentials, account locked, etc.
    console.error(err);
  } else if (mfaResponse != null) {
    plaidClient.stepAuthUser(mfaResponse.access_token, 'tomato', {},
    function(err, mfaRes, response) {
      console.log(response.accounts);
    });
  } else {
    // No MFA required - response body has accounts
    console.log(response.accounts);
  }
});
```

Chase device-based MFA flow, including using the `list:true` option to allow the user select
the device to send their security code to:
```javascript
// Add a Chase user using the list:true option
plaidClient.addConnectUser('chase', {
  username: 'plaid_test',
  password: 'plaid_good',
}, {
  list: true,
}, function(err, mfaRes, response) {
  // mfaRes.mfa is a list of send_methods
  plaidClient.stepConnectUser(mfaRes.access_token, null, {
    send_method: mfaRes.mfa[0],
  }, function(err, mfaRes, response) {
    // code was sent to the device we specified
    plaidClient.stepConnectUser(mfaRes.access_token, '1234', function(err, mfaRes, res) {
      // We now have accounts and transactions
      console.log('# transactions: ' + res.transactions.length);
      console.log('access token: ' + res.access_token);
    });
  });
});
```

Retrieve transactions for a connect user for the last thirty days:
```javascript
plaidClient.getConnectUser(access_token, {
  gte: '30 days ago',
}, function(err, response) {
  console.log('You have ' + response.transactions.length +
              ' transactions from the last thirty days.');
});
```

Associate a new webhook with a connect user (webhook PATCH):
```javascript
// Credentials are not required in this case
plaidClient.patchConnectUser(access_token, {}, {
  webhook: 'http://requestb.in',
}, function(err, mfaResponse, response) {
  // The webhook URI should receive a code 4 "webhook acknowledged" webhook
});

```

## Questions?

Open an [issue](https://github.com/plaid/plaid-node/issues/new)!

## Test

```
make test
```

Code coverage information is written to `/coverage`.

## License
[MIT](https://github.com/plaid/plaid-node/blob/master/LICENSE)
