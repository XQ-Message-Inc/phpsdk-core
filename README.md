# @xqmsg/php-core

[![version](https://img.shields.io/badge/version-0.2-green.svg)](https://semver.org)

A PHP Implementation of XQ Message SDK (V.2) which provides convenient access to the XQ Message API.

## What is XQ Message?

XQ Message is an encryption-as-a-service (EaaS) platform which gives you the tools to encrypt and route data to and from devices like mobile phones, IoT, and connected devices that are at the "edge" of the internet. The XQ platform is a lightweight and highly secure cybersecurity solution that enables self protecting data for perimeterless [zero trust](https://en.wikipedia.org/wiki/Zero_trust_security_model) data protection, even to devices and apps that have no ability to do so natively.

## Table of contents

- [@xqmsg/php-core](#xqmsgphp-core)
	- [What is XQ Message?](#what-is-xq-message)
	- [Table of contents](#table-of-contents)
	- [Installation](#installation)
		- [Prerequisites](#prerequisites)
		- [General Settings](#general-settings)
		- [Cache Configuration](#cache-configuration)
		- [Running PHPUnit Tests](#running-phpunit-tests)
		- [API Keys](#api-keys)
	- [Basic Usage](#basic-usage)
		- [Initializing the SDK](#initializing-the-sdk)
		- [Authenticating the SDK](#authenticating-the-sdk)
		- [Encrypting a Message](#encrypting-a-message)
		- [Decrypting a Message](#decrypting-a-message)
		- [Encrypting a File](#encrypting-a-file)
		- [Decrypting a File](#decrypting-a-file)
		- [Authorization](#authorization)
		- [Code Validator](#code-validator)
		- [Viewing your Access Token](#viewing-your-access-token)
		- [Revoking Key Access](#revoking-key-access)
		- [Granting and Revoking Message Access](#granting-and-revoking-message-access)
		- [Connect to an alias account](#connect-to-an-alias-account)
		- [Switching User Profiles](#switching-user-profiles)
		- [Deleting a User](#deleting-a-user)

---

## Installation

### Prerequisites

- XQ API Keys ( Generate keys at [https://manage.xqmsg.com](https://manage.xqmsg.com) )
- PHP 7.4 or higher
- Memcached 1.6 or higher.
- PHPUnit 9.5.1 or higher ( For unit tests).

### General Settings

API keys and other settings can be applied to the SDK in one of two ways:

1. **Environment Variables**: API keys can be set via PHP environment variables. The following variables are supported:

   | KEY              | DEFAULT VALUE                                                          |
   | ---------------- | ---------------------------------------------------------------------- |
   | API_KEY          | None ( Required )                                                      |
   | URL_SUBSCRIPTION | [https://subscription.xqmsg.net/v2](https://subscription.xqmsg.net/v2) |
   | URL_VALIDATION   | [https://validation.xqmsg.net/v2](https://validation.xqmsg.net/v2)     |
   | URL_QUANTUM      | [https://quantum.xqmsg.net/v2](https://quantum.xqmsg.net/v2)           |
   | ORGANIZATION_TAG | xq                                                                     |

2. **Config.php**: The Config.php file is located in the `src` directory. The environment values shown above may also be set directly within this file. If corresponding environment variables are set as well, they will override the values in this file.

### Cache Configuration

Users can configure which mechanism to use for caching user data. You can modify the default caching mechanism by editing your `Config.php` and changing the `CACHE_CLASS` value to your preferred caching implementation:

```php
// Default caching class
public const CACHE_CLASS = SessionCacheController::class;
```

- **SessionCacheController (Default)** : This uses PHP's default session management for storing user information. It is lightweight and involves no external dependencies, but data will be lost once the session is destroyed.

- **MemcachedController**: Uses memcached to manage data. Requires an accessible memcached installation.

### Running PHPUnit Tests

Run the following command from inside the main project folder to run the unit tests:

```bash
API_KEY=YOUR_API_KEY \
php /path/to/phpunit.phar --no-configuration --test-suffix php tests
```

**Note:** If the Config.php file was modified to include the keys directly, they will not need to be included above.

### API Keys

In order to utilize the XQ SDK and interact with XQ servers you will need both the **`General`** and **`Dashboard`** API keys. To generate these keys, follow these steps:

1. Go to your [XQ management portal](https://manage.xqmsg.com/applications).
2. Select or create an application.
3. Create a **`General`** key for the XQ framework API.
4. Create a **`Dashboard`** key for the XQ dashboard API.

---

## Basic Usage

### Initializing the SDK

```php
// Initialize the XQ PHP SDK
$sdk = new XQSDK();
```

**_Note: You only need to generate one SDK instance for use across your application._**

### Authenticating the SDK

Before making most XQ API calls, the user will need a required bearer token. To get one, a user must first request access, upon which they get a pre-auth token. After they confirm their account, they will send the pre-auth token to the server in return for an access token.

The user may utilize [Authorize](#authorization) to request an access token for a particular email address or telephone number, then use [CodeValidator](#code-validator) to validate and replace the temporary token they received previously for a valid access token.

Optionally the user may utilize [AuthorizeAlias](#connect-to-an-alias-account) which allows them to immediately authorize an access token for a particular email address or telephone number. It should only be considered in situations wher 2FA is problematic/unecessary, such as IoT devices or trusted applications that have a different method for authenticating user information.

**_Note: This method only works for new email addresses/phone numbers that are not registered with XQ. If a previously registered account is used, an error will be returned._**

### Encrypting a Message

The most straightforward way to encrypt a mesage is by using the `Encrypt.runWith` method:

```php

// The message to encrypt
$msg = "Hello World";

// The message recipients
$recipients = "john@email.com";

$response = Encrypt::with($sdk)->runWith(
	"Hello World", // The message to encrypt
  "john@email.com", // The message recipients
  new AESAlgorithm(),  // The encryption algorithm
  24, // The number of hours before the message expires
  [
  	'subject' => 'Sample Message',  // Additional dashboard metadata
  ]
);

if (!$response->succeeded()){
	// Something went wrong...
	echo . $response->status();
	die();
}


// Encryption was successful.
// The data can be used in any fashion the user likes.
// Below, it is URL encoded for transmission as an email.
$encrypted = $response->json();
$url = $sdk->encodeLink($encrypted->data, $encrypted->token);

echo "Encrypted URL: " . $url;

```

See the `Encrypt.php` file for more details and available options when encrypting.

### Decrypting a Message

Encrypted text can be decrypted via `Decrypt.run` or `Decrypt.runWith`:

```php

// Assuming the message was received as a url,
// it would first need to be decoded.

$encrypted = $sdk->decodeLink($url);

$response = Decrypt::with($sdk)->run( (array) $encrypted );

if (!$response->succeeded()){
	// Something went wrong...
	echo . $response->status();
	die();
}

echo "Decrypted Content: " . $response->raw();
```

### Encrypting a File

Files can be encrypted via `EncryptFile.run` or `EncryptFile.runWith`.

```php

// Create a file "upload" using a local file.
$filename = "sample.txt";
$sourcePath = __DIR__ . "/" . $filename;
$source = File::uploaded([
	"name" => $filename,
	"tmp_name" => $sourcePath,
	"type" => mime_content_type($sourcePath),
	"error" => UPLOAD_ERR_OK,
	"size" => filesize( $sourcePath )
]);

// This is where our encrypted file will be saved in the filesystem.
// If this is not provided, the result will returned and not saved.
$targetPath =  __DIR__ . "/Sample-encrypted.txt.xqf";

// The number of hours before the key expires.
$expiresIn = 48;

$response = EncryptFile::with($sdk)->runWith(
$source,
$targetPath,
$recipients,
new OTPv2Algorithm(),
$expiresIn );

if (!$response->succeeded()){
	// Something went wrong...
	echo . $response->status();
	die();
}

// Success - The encrypted file should be accessible at $targetPath.

```

See the `EncryptFile.php` file for more details and available options when encrypting.

### Decrypting a File

Encrypted files can be decrypted via `DecryptFile.run` or `DecryptFile.runWith`.

```php

$filename = "sample.txt.xqf";
$sourcePath = __DIR__ . "/" . $filename;


// Create a file "upload" using a local file.
 $source = File::uploaded([
 	"name" => $filename,
 	"tmp_name" => $sourcePath,
 	"error" => UPLOAD_ERR_OK,
 	"size" => filesize( $targetPath )
 ]);

 // This is where our decrypted file will be saved in the filesystem.
// If this is not provided, the result will returned and not saved.
$targetPath =  __DIR__ . "/sample.txt";


 $response = DecryptFile::with($sdk)->runWith( $source, $targetPath );

 if (!$response->succeeded()){
	// Something went wrong...
	echo . $response->status();
	die();
}

// Success - The decrypted file should be accessible at $targetPath.

```

### Authorization

Request an access token for a particular email address. If successful, the user will receive an email containing a PIN and a validation link.

The service itself will return a pre-authorization token that can be exchanged for a full access token once validation is complete.

```php

$arguments = [
	'user' => "john@email.com"
];

$response = RequestAccess::with($sdk)->run( $arguments );
if (!$response->succeeded()){
	// Something went wrong...
	echo . $response->status();
	die();
}
// Success - A preauthorization token will be stored in memory.
// This will be replaced with a real authorization token once the user
// has successfully validated their email address by clicking on the link  // they received.
```

### Code Validator

After requesting authorization via the `RequestAccess` class, the user can submit the PIN they received using the `ValidateAccessRequest` class. Once submitted and validated, the temporary token they received previously will be replaced for a valid access token that can be used for other requests:

```php
$arguments = [
	'code' => "123456"
];

$response = ValidateAccessRequest::with($sdk)->run($arguments);
if (!$response->succeeded()){
	// Something went wrong...
	echo . $response->status();
	die();
}

// Success - User is Authorized.
```

Alternatively, if the user clicks on the link in the email, they can simply exchange their pre-authorization token for a valid access token by using the Exchange class directly. Note that the active user in the sdk should be the same as the one used to make the authorization call:

```php
$response = Exchange::with($sdk)->run();
if (!$response->succeeded()){
	// Something went wrong...
	echo . $response->status();
	die();
}

// Success - User is Authorized.
```

### Viewing your Access Token

An access token can be stored out-of-band and reused for as long as it is valid. If this is done, ensure that it is stored in a manner that only authorized users have access to it.

```php

// Reference the cache instance.
$cache = $sdk->getCache();

// Get the active user profile:
$activeProfile = $cache->getActiveProfile();

// Get the XQ access token ( if available ):
$access_token = $cache->getXQAccess($activeProfile);

// Restore an access token:
$cache->addXQAccess($activeProfile, access_token );
```

### Revoking Key Access

Access to an entire message can also be revoked. When this occurs, both the sender and all the recipients will lose all access to it. **Note that this action is not reversible**:

```php
 // Revoke all access to key.
$response = RevokeKey::with($sdk)->runWith( $encryptedObject->token );

 if (!$response->succeeded()){
	// Something went wrong...
	echo . $response->status();
	die();
}

// Success - Message has been revoked.
```

### Granting and Revoking Message Access

There may be cases where a user needs to grant or revoke access to a previously sent message. This can be achieved via `GrantKeyAccess` and `RevokeKeyAccess` respectively:

```php

// The token of the previously existing message.
$token = "MY_MESSAGE_TOKEN"
// Add jim and joe to the recipient list.
$emailsToAdd = array("jim@email.com","joe@email.com");

// Grant access to jim@email.com. This will only work if the currently
// active profile is the same one that sent the original message.
$response = GrantKeyAccess::with($sdk)->runWith($token, $emailsToAdd );

 if (!$response->succeeded()){
	// Something went wrong...
	echo . $response->status();
	die();
}

// Success - Jim and Joe will now be able to read the original message.

// Revoke access for Joe alone.
$emailsToRemove = array("joe@email.com");
$response = RevokeKeyAccess::with($sdk)->runWith($token, $emailsToRemove );

 if (!$response->succeeded()){
	// Something went wrong...
	echo . $response->status();
	die();
}

// Success - Joe's access to this message has been revoked.
```

### Connect to an alias account

After creation, a user can connect to an Alias account by using the AuthorizeAlias endpoint:

```php
$arguments = [
	'user' => "john@email.com"
];

$response = RequestAliasAccess::with($sdk)->run( $arguments );
if (!$response->succeeded()){
	// Something went wrong...
	echo . $response->status();
	die();
}
// Success - The alias user was authorized.

```

<!-- TODO: Add Quantum Entropy -->
<!-- TODO: Add Dashboard Management -->

### Switching User Profiles

Authorized users have their access token cached. This allows them to switch between different profiles, and potentially start and stop the application without needing to reauthorize every time.

```php

// Assuming a user previously logged on with Jane's email account and
// want to use it for any upcoming actions ( like sending an email):

// 1. Save your active profile:
$activeProfile = $sdk->getCache()->getActiveProfile();

// Switch to Jane's profile
$sdk->switchProfile("jane@email.com");

// After performing actions as Jane, switch back to previous profile:
$sdk->switchProfile($activeProfile);

```

### Deleting a User

Authenticated users can be deleted via `DeleteUser.run` or `DeleteUser.runWith`. In order to continue use after deleting a user, it will be necessary to switch to another active user profile (if one is available), or request access for a new user.

```php

// Delete the active user.
$response = DeleteUser::with($sdk)->run();

if (!$response->succeeded()){
	// Something went wrong...
	echo . $response->status();
	die();
}

// Success - User has been successfully deleted.

```
