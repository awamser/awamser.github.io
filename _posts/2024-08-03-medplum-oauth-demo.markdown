---
layout: post
title: "Medplum OAuth Demo with SwiftUI"
date: 2024-08-03 17:47:10 -0500
categories: medplum oauth
---

In this demo, I'll demonstrates `OAuth` authentication with `Medplum` in a SwiftUI iOS application. The app will allow a user to log in using their Medplum credentials and display their access token upon successful authentication (you should never show
your token ;). The focus is on the AuthViewModel class, which handles the OAuth flow and token retrieval.

## Medplum OAuth Flow

The following [link](https://www.medplum.com/docs/auth/methods/oauth-auth-code) provides a detailed explanation of the OAuth 2.0 authorization code flow for Medplum:

## AuthViewModel Class

All of our OAuth implementation is in the `AuthViewModel` class. This class manages the authentication state, handles the OAuth flow, and provides methods for logging in and out. Let's break down its key components:

### Key Properties

- `isAuthenticated`: A boolean indicating whether the user is currently authenticated.
- `accessToken`: Stores the access token received after successful authentication.
- `error`: Stores any error messages that occur during the authentication process.
- `clientId`: The client ID for your Medplum application.
- `redirectUri`: The URI to which Medplum will redirect after authentication.
- `codeVerifier` and `codeChallenge`: Used for the PKCE extension of OAuth 2.0.

### Login Function

The `login()` function initiates the OAuth flow:

- It generates a code verifier and challenge for PKCE.
- Constructs the authorization URL with necessary parameters.
- Starts an `ASWebAuthenticationSession` for secure, in-app authentication.

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

ASWebAuthenticationSession is an Apple-provided class in the AuthenticationServices Framework that manages web-based authentication sessions. It's useful for OAuth flows because it:

- Provides a secure, in-app web view for user authentication.
- Handles callback URLs automatically.
- Ensures privacy by using isolated sessions:

## Generating authURL and callbackURLScheme.

Creating the authURL is straightforward based on the Medplum documentation. Important: When calling the authorize endpoint with `grant_type` set to `authorization_code`, you must include the `code_challenge` and `code_challenge_method`. These parameters are required for PKCE (Proof Key for Code Exchange) and is necessary for requesting the token later.

### Create the code challange for PKCE

The `generateCodeVerifier` function create and stores the `codeVerifier` and `codeChallenge` required for OAuth 2.0 PKCE, which enhances security for public clients like mobile apps that can’t securely store a client secret. This flow ensures robust and secure authentication, protecting both
user credentials and access tokens.

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

### Generate the `authURL`

Now that we have the `code_challenge` and `code_verifier`, we can create the authURL. The authURL is the URL to which the user will be redirected to for authentication. The URL includes the client ID, response type, redirect URI, scope, and code
challenge.

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

### Generate the `callbackURLScheme`

The `callbackURLScheme` is a custom URL scheme that the app registers in its Info.plist file. It's used to return the authorization code to the app after the user completes the authentication process. You can add the redirect URL in the Project -->
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

### Extract the scheme from the redirect URL:

{% highlight swift %}

let scheme = URL(string: redirectUri)!.scheme

{% endhighlight %}

### Create ASWebAuthenticationSession and handle the callback

The code creates an ASWebAuthenticationSession with the authorization URL and callback scheme URL. When the user completes the authentication, the system captures the callback URL and returns it to our app, where you extract the authorization code.
Once you have the authorization code, you can exchange it for an access token with a simple POST call to the Medplum token [endpoint](https://www.medplum.com/docs/api/oauth/token).

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

After receiving the authorization code via the `handleCallback()` function, the `exchangeCodeForToken()` function is called to obtain the access token:

{% highlight swift %} 

    private func handleCallback(_ url: URL) -> String? {
        guard let urlComponents = URLComponents(url: url, resolvingAgainstBaseURL: false) else {
            return nil
        }
        return urlComponents.queryItems?.first(where: { $0.name == "code" })?.value
    }

{% endhighlight %}

This function sends a POST request to Medplum's token endpoint with the necessary parameters, including the authorization `code` and `code_verifier`. Upon successful response, it updates the `accessToken` and `isAuthenticated` properties.

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

## Conclusion

This `AuthViewModel` provides a robust implementation of OAuth 2.0 authentication with PKCE for Medplum. By using `ASWebAuthenticationSession`, it ensures a secure authentication flow within your SwiftUI app. Remember to handle the authentication
state in your UI, displaying login options when `isAuthenticated` is false and secured content when it's true.

For a complete implementation, you'll need to create SwiftUI views that utilize this view model and respond to changes in the authentication state. Always ensure you're following best practices for securing tokens and handling user data in your production
applications. Complete source code for this implementation is available on [GitHub](https://github.com/awamser/MedplumSwiftUIAuth)
