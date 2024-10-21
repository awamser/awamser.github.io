---
layout: post
title: "Medplum OAuth Demo with SwiftUI"
date: 2024-10-03
categories: medplum oauth
---

This guide provides instructions on setting up OAuth authentication in a native SwiftUI iOS application to authenticate with Medplum. OAuth is a widely adopted open standard that enables secure access delegation. It allows third-party applications to request access to a user’s resources, such as their account information, data, or files, without needing to obtain and store the user’s credentials (e.g., username and password).

OAuth provides a token representing the user’s consent to grant specific access rights to the requesting application. The application can use this token to perform actions on behalf of the user, such as accessing a protected API or retrieving user data, without compromising the user’s security.

In this guide, we’ll explore how to leverage OAuth with Medplum using SwiftUI and the ASWebAuthenticationSession framework. This will allow you to authenticate users and securely access their Medplum data within your iOS application.

### Medplum OAuth Flow

The following [link](https://www.medplum.com/docs/auth/methods/oauth-auth-code) provides details of the OAuth 2.0 authorization code flow for Medplum.

### AuthViewModel Class

All of the OAuth implementation is in the `AuthViewModel` class. This class manages the authentication state, handles the OAuth flow, and provides methods for logging in and out.

### Key Properties

- `isAuthenticated`: A boolean indicating whether the user is currently authenticated.
- `accessToken`: Stores the access token received after successful authentication.
- `error`: Stores any error messages that occur during the authentication process.
- `clientId`: The client ID for the Medplum application.
- `redirectUri`: The URI to which Medplum will redirect after authentication.
- `codeVerifier` and `codeChallenge`: Used for the PKCE extension of OAuth 2.0.

### Login Function

The `login()` function initiates the OAuth flow, it generates a code verifier and challenge for PKCE, constructs the authorization URL with necessary parameters,and starts an `ASWebAuthenticationSession` for secure, in-app authentication.

 
{% highlight swift %}

func login() {
 
    generateCodeVerifier()

    guard let authUrl = createAuthURL() else {
        handleError(.invalidURLComponents)
        return
    }

    let scheme = URL(string: AuthConfig.redirectUri)!.scheme

    startAuthSession(authUrl: authUrl, scheme: scheme)

}

{% endhighlight %}

### OAuth Authentication using ASWebAuthenticationSession

[ASWebAuthenticationSession](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession) is an Apple-provided class in the AuthenticationServices Framework that manages web-based authentication sessions. It’s useful for OAuth flows because it; provides a secure, in-app web view for user authentication, handles callback URLs automatically, and ensures privacy using isolated sessions.

The ASWebAuthenticationSession initializer requires an `authURL` and `schema`.

### Generating authURL and scheme (redirectUri)

Creating the authURL is straightforward using the [Medplum documentation](https://www.medplum.com/docs/auth/methods/oauth-auth-code#authorize-your-client). Important: When calling the authorize endpoint with `grant_type` set to `authorization_code`, is to include the `code_challenge` and `code_challenge_method`. These parameters are required for PKCE (Proof Key for Code Exchange) and necessary for requesting the token later.

### Create the code challange for PKCE

The `generateCodeVerifier()` function creates and stores the `codeVerifier` and `codeChallenge` required for OAuth 2.0 PKCE, which enhances security for public clients like mobile apps that can’t securely store a client secret. This flow ensures robust and secure authentication, protecting user credentials and access tokens.

{% highlight swift %}
private func generateCodeVerifier() {
    var buffer = [UInt8](repeating: 0, count: 32)
    _ = SecRandomCopyBytes(kSecRandomDefault, buffer.count, &buffer)

    codeVerifier = Data(buffer).base64EncodedString()
        .replacingOccurrences(of: "+", with: "-")
        .replacingOccurrences(of: "/", with: "_")
        .replacingOccurrences(of: "=", with: "")
        .trimmingCharacters(in: .whitespaces)

    guard let verifier = codeVerifier else {
        handleError(.unknownError)
        return
     }

    let challengeHash = SHA256.hash(data: Data(verifier.utf8))
    codeChallenge = Data(challengeHash)
        .base64EncodedString()
        .replacingOccurrences(of: "+", with: "-")
        .replacingOccurrences(of: "/", with: "_")
        .replacingOccurrences(of: "=", with: "")
        .trimmingCharacters(in: .whitespaces)
}

{% endhighlight %}

### Create the `authURL`

With the `code_challenge` and `code_verifier`, create the authURL. The authURL is the URL to which the user will be redirected to for authentication. It includes the client ID, response type, redirect URI, scope, and code
challenge details. The `loadClientId()` reads the clientId from the Secrets.plist, create this file (Property List) in Xcode.

**Note**: Create a [Client Appliation](https://www.medplum.com/docs/api/fhir/medplum/clientapplication) in Medplum for the clientId and also to configure the Redirect URI (`medplum-oauth://redirect`) described next.

### Create the `scheme URI`

The `scheme` is a custom URL that the app registers in its Info.plist file. It returns the authorization code to the app after the user completes the authentication process. Add the redirect URL in the Project -->
Info --> URL Types section in Xcode or manually.

{% highlight xml %}

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>CFBundleURLTypes</key>
    <array>
        <dict>
            <key>CFBundleTypeRole</key>
            <string>Viewer</string>
            <key>CFBundleURLName</key>
            <string>com.[your-domain].redirect</string>
            <key>CFBundleURLSchemes</key>
            <array>
                <string>medplum-oauth</string>
            </array>
        </dict>
    </array>
</dict>
</plist>

{% endhighlight %}

### Extract the scheme from the redirect URL

{% highlight swift %}

let scheme = URL(string: redirectUri)!.scheme

{% endhighlight %}

### Create URL required by ASWebAuthenticationSession

Apple documentation for [ASWebAuthenticationSession Initializer](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession#Initializers).

{% highlight swift %}

private func createAuthURL() -> URL? {
    var components = URLComponents(string: "\(AuthConfig.baseUrl)\(AuthConfig.authorizeEndpoint)")
    components?.queryItems = [
        URLQueryItem(name: "client_id", value:  loadClientId()),
        URLQueryItem(name: "response_type", value: AuthConfig.responseType),
        URLQueryItem(name: "redirect_uri", value: AuthConfig.redirectUri),
        URLQueryItem(name: "scope", value: AuthConfig.scope),
        URLQueryItem(name: "code_challenge", value: codeChallenge),
        URLQueryItem(name: "code_challenge_method", value: AuthConfig.codeChallengeMethod)
    ]
    return components?.url
}

{% endhighlight %}

### Create ASWebAuthenticationSession and handle the callback

Creates an ASWebAuthenticationSession with the authorization URL and callback scheme. When the user completes the authentication, the system captures the callback URL and returns it to the app, and extract the authorization code.
Once the authorization code is returned, exchange it for an access token with a simple POST call to the Medplum token [endpoint](https://www.medplum.com/docs/api/oauth/token).

{% highlight swift %}

private func startAuthSession(authUrl: URL, scheme: String?) {
    let session = ASWebAuthenticationSession(url: authUrl, callbackURLScheme: scheme) { callbackURL, error in
        if let error = error {
            self.handleError(.authenticationFailed(error.localizedDescription))
            return
        }

        guard let callbackURL = callbackURL, let code = self.handleCallback(callbackURL) else {
            self.handleError(.authorizationCodeParsingFailed)
            return
        }

        self.exchangeCodeForToken(code)
    }

    session.presentationContextProvider = self
    session.start()
}

{% endhighlight %}

### Exchanging the Code for a Token

Receive the authorization code via the `handleCallback()` function in the startAuthSession, then the `exchangeCodeForToken()` function obtains the access token.

{% highlight swift %}

private func handleCallback(_ url: URL) -> String? {
    guard let urlComponents = URLComponents(url: url, resolvingAgainstBaseURL: false) else {
        return nil
    }
    return urlComponents.queryItems?.first(where: { $0.name == "code" })?.value
}

{% endhighlight %}

The `exchangeCodeForToken()` function sends a POST request to Medplum's token endpoint with the necessary parameters, including the authorization `code` and `code_verifier`. Upon successful response, it updates the `accessToken` and `isAuthenticated` properties.

{% highlight swift %}

private func exchangeCodeForToken(_ code: String) {
    guard let tokenUrl = URL(string: "\(AuthConfig.baseUrl)\(AuthConfig.tokenEndpoint)") else {
        handleError(.invalidURLComponents)
        return
    }

    var request = URLRequest(url: tokenUrl)
    request.httpMethod = "POST"
    request.setValue("application/x-www-form-urlencoded", forHTTPHeaderField: "Content-Type")

    let bodyParams = [
        "grant_type": "authorization_code",
        "client_id":  loadClientId(),
        "code": code,
        "redirect_uri": AuthConfig.redirectUri,
        "code_verifier": codeVerifier ?? ""
    ]

    request.httpBody = bodyParams.map { "\($0.key)=\($0.value)" }.joined(separator: "&").data(using: .utf8)

    URLSession.shared.dataTask(with: request) { data, response, error in
        DispatchQueue.main.async {
            if let error = error {
                self.handleError(.networkError(error.localizedDescription))
                return
            }

            guard let data = data else {
                self.handleError(.unknownError)
                return
            }

            self.handleTokenResponse(data: data)
        }
    }.resume()
}

{% endhighlight %}

### Conclusion

This AuthViewModel robustly implements OAuth 2.0 authentication with PKCE for Medplum. By using ASWebAuthenticationSession, it ensures a secure authentication flow within a SwiftUI app. Remember to handle the authentication state in the UI, displaying login options when isAuthenticated is false and secured content when it’s true.

For a complete implementation, create SwiftUI views that utilize this view model and respond to changes in the authentication state. Always following best practices for securing tokens and handling user data in production
applications. Complete source code for this implementation is available on [GitHub](https://github.com/awamser/MedplumSwiftUIAuth)
