# Google Authentication

As FreeLeh implementations depend on Google Services and APIs, we need to understand how to prepare relevant
information related to the Google Authentication flow.

## OAuth2 Flow

This flow follows the normal [OAuth2 Flow](https://developers.google.com/identity/protocols/oauth2).
Note that different FreeLeh language implementation supports different subsets of Google OAuth2 flows.

1. `GoFreeDB` only supports the server-side OAuth2 flow.
2. `PyFreeDB` only supports the server-side OAuth2 flow.

### Server Side Flow

There are 3 main information required for server side flow:

1. A JSON file containing the `client_secret`. 
    - Create a new OAuth2 client secret JSON via [Google Developers Console](https://console.cloud.google.com/apis/credentials).
    - You can put any link for the redirection URL field.


2. A target JSON file path in which relevant credentials information will be written into.
    - After the authentication is done, relevant credentials information will be written into this file **automatically**.
    - This file will contain the access token and refresh token.
    - You can think of this as a cache of the credentials information so the library does not have to keep triggering the full OAuth2 flow.

   
3. A list of `scopes` to tell Google what kind of resource permissions the OAuth2 flow should be requesting. 
    - Each FreeLeh project provides a constant defining what scopes are required. You can refer to each project for more details.

During the OAuth2 flow, you will be asked to click a generated URL in the terminal.

1. Click the link and authenticate your Google Account.
2. You will eventually be redirected to another link which contains the authentication code in the URL (not the access token yet).
3. Copy and paste that final redirected URL (from the browser) into the terminal to finish the flow.

#### Golang

```go
import "github.com/FreeLeh/GoFreeDB/google/auth"

auth, err := auth.NewOAuth2FromFile(
    "<path_to_client_secret_json>",
    "<path_to_cached_credentials_json>",
    scopes,
    auth.OAuth2Config{},
)
```

#### Python

```python
from pyfreedb.providers.google.auth import OAuth2GoogleAuthClient


cached_credentials_info = {
    "token": "token",
    "refresh_token": "refresh_token",
    "token_uri": "token_uri",
    "client_id": "client_id",
    "client_secret": "client_secret",
    "scopes": ["https://www.googleapis.com/auth/spreadsheets"],
    "expiry": "2022-08-23T13:21:35.408789",
}

# If client has the cached credentials information, you can pass that information directly.
auth = OAuth2GoogleAuthClient.from_authorized_user_info(cached_credentials_info, scopes)


# If client has not finished the OAuth2 flow, you can use the downloaded client secret file to start the OAuth2 flow.
# Note that this will create the cached credentials JSON file with a content like `cached_credentials_info` dictionary above.
auth = OAuth2GoogleAuthClient.from_authorized_user_file(
   "<path_to_cached_credentials_json>",
   "<path_to_client_secret_json>",
   scopes,
)
```

## Service Account Flow

This flow follows the [Google Service Account flow](https://developers.google.com/identity/protocols/oauth2/service-account).
This flow is very useful if you have a script running independently (no frontend, no server).

There are 2 main information required for the service account flow:

1. A service account credentials JSON file.
   - Create a new service account credentials via [Google Service Account page](https://developers.google.com/identity/protocols/oauth2/service-account#creatinganaccount).
   - Create a new service account key.
   - Download the credentials JSON file for that new service account key.


2. A list of `scopes` to tell Google what kind of resource permissions the OAuth2 flow should be requesting.
   - Each FreeLeh project provides a constant defining what scopes are required. You can refer to each project for more details.

> ### ⚠️ ⚠️ Warning
> Note that a service account is just like an account.
> The email in the `service_account_json` must have required accesses to the target Google resources
> (e.g. read write access to the target Google Sheets) just like a normal email address.
> Otherwise, you will get an authorization error.

#### Golang

```go
import "github.com/FreeLeh/GoFreeDB/google/auth"

// Use this if you have the service account credentials in JSON file format.
auth, err := auth.NewServiceFromFile(
    "<path_to_service_account_json>",
    scopes,
    auth.OAuth2Config{},
)


// Use this if you have the service account credentials in raw JSON bytes format.
auth, err := auth.NewServiceFromRaw(
   rawServiceAccountCredentialsJSONBytes,
   scopes,
   auth.OAuth2Config{},
)
```

#### Python

```python
from pyfreedb.providers.google.auth import ServiceAccountGoogleAuthClient


# Use this if you have the service account credentials in JSON file format.
auth = ServiceAccountGoogleAuthClient.from_service_account_file(
   "<path_to_service_account_json>",
   scopes,
)


# Use this if you have the service account credentials in dictionary format.
auth = ServiceAccountGoogleAuthClient.from_service_account_info(
   service_account_credentials_dict,
   scopes,
)
```