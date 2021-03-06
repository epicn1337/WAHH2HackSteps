CHAPTER 7: ATTACKING SESSION MANAGEMENTY

In many applications that use the standard cookie mechanism to transmit session tokens, it is straightforward to identify which item of data contains the token. However, in other cases this may require some detective work.
1. The application may often employ several different items of data collectively as a token, including cookies, URL parameters, and hiden form fields. Some of these items may be used to maintain session state on different back-end components. Do not assume that a particular parameter is the session token without proving it, or that sessions are being tracked using only one item.
2. Sometimes, items that appear to be the application's session token may not be. In particular, the standard session cookie generated by the web server or application platform may be present but not actually used by the application.
3. Observe which new items are passed to the browser after authentication. Often, new session tokens are created after a user authenticates themself.
4. To verify which items are actually being employed as tokens, find a page that is definitely session-dependent (such as user-specific "my details" page). Make several requests for it, systematically removing each item that you suspect is being used as a token. If removing an item causes the session-dependent page not to be returned, this *may* confirm that the item is a session token. Burp repeater is a useful tool for performing these tests.

## Alternatives to Sessions:
1. If HTTP authentication is being used, it is possible that no session management mechanism is implemented. Use the methods described previously to examin the role played by any token-like items of data.
2. If the application uses a sessionless state mechanism, transmitting all data required to maintain state via the client, this may sometimes be difficult to detect with certainty,but the following are strong indicators that this kind of mechanism is being used:
	- Token-like data items issued to the client are fairly long (100 or more bytes).
	- The application issues a new token-like item in response to every request.
	- The data in the item appears to be encrypted (and therefore has no discernible structure) or signed (and therefore has a meaniful structure accompanied by a few bytes of meaninless binary data)
	- The application may reject attempts to submit the same item with more than one request.
3. If the evidence suggests strongly that the application is not using session tokens to manage state, it is unlikely that any of the attacks described in this chapter will accomplish anything. Spend time looking for other serious issues like broken access and/or code injection

## Weaknesses in Token Generation:
1. Obtain a single token from the application, and modify it in systematic ways to determine whether the entire token is validated or whether some of its subcomponents are ignored. Try changing the token's value one byte at a time (or even one bit at a time) and resubmitting the modified token to the application to determine whether it is still accepted. If you find that certain portions of the token are not actually required to be correct, you can exclude these from any further analysis, potentially reducing the amount of work you need to perform. You can use the "char frobber" payload type in Burp Intruder to modify a token's value in one character position at a time, to help with this task.
2. Log in as several different users at different times, and record the tokens received from the server. If self-registration is available and you can choose your username, log in with a series of similar usernames containing small variations between them, such as A,AA,AAA,AAAA,AAAB,AAAC,AABA and so on. If other user-specific data is submitted at login or stored in user profiles (such as an email-address) perform a similar exercise to vary that data systematically and record the tokens received following login.
3. Analyze the tokens for any correlations that appear to be related to the username and other user-controllable data.
4. Analyze the tokens for any detectable encoding or obfuscation. Where the username contains a sequence of the same character, look for a corresponding character sequence in the token, which may indicate the use of XOR obfuscation. Look for sequences in the token containing only hexadecimal characters, which may indicate a hex encoding of an ASCII string or other information. Look for sequences that end in an equals sign and/or that contain only the other valid Base64 characers: a to z, A to Z, 0 to 9, + and /
5. If any meaning can be reverse-engineered from the sample session tokens, consider whether you have sufficient information to attempt to guess the tokens recently issued to other application users. Find a page of the application that is session-dependent, such as one that returns an error message or redirect elsewhere if accessed without valid session. Then use Burp Intruder to make large numbers of requests to this page using guessed tokens. Monitor the results for any cases in which the page loaded correctly, indicating a valid session token.

## Hack Steps:
1. Determine when and how session tokens are issued by walking through the application from the first application page through any login functions. Two behaviors are common:
	- The application creates a new session anytime a request is received that does not submit a token
	- The application creates a new session following a successful login.
- To harvest large numbers of tokens in an automated way, ideally identify a single request (typically a GET / or a login submission) that causes a new token to be issued.
2. In Burp, send the request that creates a new session to Sequencer, and configure token's location. Then start a live capture to gather as many tokens as feasible. If a custom session management mechanism is in use, and you only have remote access to the application, gather the tokens as quickly as possible to minimize the loss of tokens issued to other users and reduce the influence of any time dependency.
3. If a commercial session management mechanism is in use and/or you have local access to the application, you can obtain indefinitely large sequences of session tokens in controlled conditions.
4. While Burp Sequencer is capturing tokens, enable the "auto analyze" setting to that Burp automatically performs the statistical analysis periodically. Collect at least 500 tokens before reviewing the results in any detail. If a sufficient number of hits within the token have passed the tests, continue gathering tokens for as long as feasible, reviewing the analysis results as further tokens are captured.
5. If the tokens fail the randomness tests and appear to contain patterns that could be exploited to predict future tokens, reperform the exercise from a different IP address and (if relevant), a different username. This will help you identify whether the same pattern is detected and whether tokens received in the first exercise could be extrapolated to identify tokens received in the second. Sometimes the sequence of tokens captured by one user manifests a pattern. But this will not allow straight-forward extrapolation to the tokens issued to other users, because information such as source IP is used as a source of entropy (such as a seed to a random number generator).
6. If you believe you have enough insight into the token generation algorithm to mount an automated attack against other users' sessions, it is likely that the best means of achieving this is via a cusomized script. This can generate tokens using the specific patterns you have observed and apply any necessary encoding. See Chapter 15 for some generic techniques for applying automation to this type of problem.
7. If source code is available, closely review the code responsible for generating session tokens to understand the mechanism used and determine whether it is vulnerable to prediction. If entropy is drawn from data that can be determined within the application within a brute-forcible range, consider the practical number of requests that would be needed to brute-force an application token.

## Encrypted Tokens:
In many situations where encrypted tokens are used, actual exploitability may depend on various factors, including the offsets of block bounderies relative to the data you need to attack, and the application's tolerance of the changes that you cause to the surrounding plaintext structure. Working completely blind, it may appear difficult to construct an effective attack, however in many situations this is in fact possible.
1. Unless the session token is obviously meaninful or sequential in itself, always consider the possibility that it might be encrypted. You can often identify that a block-based cipher is being used by registering several different usernames and adding one character in length each time. If you find a point where adding one character results in your session token jumping in length by 8 or 16 bytes, then a block cipher is probably being used. You can confirm this by continuing to add bytes to your username and looking for the same jump occuring 8 or 16 bytes later.
2. ECB cipher manipulation vulnerabilites are normally difficult to identify and exploit in a purely black-box context. You can try blindly duplicated and moving the ciphertext blocks within your token, and reviewing whether you remain loggin in to the application within your own user context, or that of another user, or none at all.
3. You can test for CBC cipher manipulation vulnerabilites by running a Burp Intruder attack over the whole token, using the "bit flipping" payload source. If the bit flipping attack identifies a section within the token, the manipulation of which causes you to remain in a valid session, but as a different or nonexistent user, perform a more focused attack on just this section, trying a wider range of values at each position.
4. During both attacks, monitor the application's responses to identify the user associated with your session following each request, and try to exploit any opportunities for privilege escalation that may result.
5. If you attacks are unsuccessful, but it appears from step 1 that variable-length input that you control is being incorporated into the token, you should try generating a series of tokens by adding one character at a time, at least up to the size of blocks being used. For each tesulting token, you should reperform steps 2 and 3. This will increase the chance that the data you need to modify is suitably aligned with block bounderies for your attack to succeed.
# Weaknessess in Session Token Handling
## Disclosure of Tokens on the Network:
1. Walk through the application in the normal way from first access (the "start" URL), through the login process, and then through all of the application's functionality. Keep a record of every URL visited, and note every instance in which a new session token is received. Pay particular attention to login functions and transitions between HTTP and HTTPS communications. This can be achieved manually using a network sniffer such as wireshark or partially automated using the loggin function of Burp.
2. If HTTP cookies are being used as the transmission mechanism for session tokens, verify whether the **secure** flag is set, preventing them from ever being transmitted over unencrypted connections.
3. Determine whether, in the normal use of the application, session tokens are every transmitted over an unencrypted connection. If so, they should be regarded as vulnerable to interception.
4. Where the start page uses HTTP, and the application switches to HTTPS for the login and authenticated areas of the site, verify whether a new token is issued following the login, or whether a token transmitted during the HTTP stage is still being used to track the user's authenticated session. Also, verify whether the application will accept login over HTTP if the login URL is modified accordingly. 
5. Even if the application uses HTTPS for every page, verify whether the server is also listening on port 80, running any service or content. If so, visit any HTTP URL directly from wihin an authenticated session and verify whether the session token is transmitted.
6. In cases where a token for an authenticated session is transmitted to the server over HTTP, verify whether that token continues to be valid or is immediatedly terminated by the server.
## Disclosure of Tokens in Logs:
1. Identify all the functionality within the application, and locate any logging or monitoring functions where session tokens can be viewed. Verify who can access this functionality - for example, administrators, any authenticated user, or any anonymous user. See Chapter 4 for techniques for discovering hidden content that is not directly linked from the main application.
2. Identify any instances within the application where session tokens are transmitted within the URL. It may be that tokens are generally transmitted in a more secure manner but that the developers have used the URL in specific cases to work around particular difficulties. For example, this behavior is often observed where a web application interfaces with an external system.
3. If session tokens are being transmitted in URLs, attempt to find any application functionality that enables you to inject arbitrary off-site links into pages viewed by other users. Examples include functionality implementing a message board, site feedback, question-and-answer, and so on. If so, submit links to a web server that you control and wait to see whether any users' session tokens are recieved in your Referer logs.
4. If any session tokens are captured, attempt to hijack user session by using the application as normal but substituting a captured token for your own. You can do this by intercepting the next response from the server and adding a **Set-Cookie** header of your own with the captured cookie value. In Burp, you can apply a single Suite-wide config that sets a specific cookie in all request to the target application to allow easy switching between different session contexts during testing.
5. If a large number of tokens are captured, and session hijacking allows you access sesitive data such as personal details, payment info, or user passwords, you can use the automated techniques described in Chapter 14 to harvest all desired data belonging to other application users.
## Vulnerable Mapping of Tokens to Sessions:
1. Log in to the application twice using the same user account, either from different browser processes or from different computers. Determine whether both sessions remain active concurrently. If so, the application supports concurrent sessions, enabling an attacker who has compromised another user's credentials to make use of these without risk of detection.
2. Log in and log out several times using the same user account, either from different browser processes or from different computers. Determine whether a new session token is issued each time or whether the same toke is issued each time you log in. If the latter occurs, the application is not really employing proper sessions.
3. If tokens appear to contain any structure and meaning, attempt to seperate out components that may identify the user from those that appear to be inscrutable. Try to modify any user-related components of the token so that they refer to other known users of the application, and verify whether the resulting token is accepted by the application and enables you to masquerade as that user.
## Vulnerable Session Termination:
1. Do not fall into the trap of examining actions that the application performs on the client-side token (such as cookie invalidation via a new `Set-cookie` instruction, client-side script, or an expiration time attribute). In terms of session termination, nothing much depends on what happens to the token within the client browser. Rather, investigate whether session expiration is implemented on the server side:
	- Log in to the application to obtain a valid session token.
	- Wait for a period without using this token, and then submit a request for a protected page (such as "my details") using the token.
	- If the page is displayed as normal, the token is still active.
	- Use trial and error to determine how long any session expiration time-out is, or whether a token can still be used days after the last request using it. Burp Intruder can be configured to increment the time interval between successive requests to automate this task.
2. Determine whether a logout function exists and is prominently made available to users. If not, users are more vulnerable, because they have no way to cause the application to invalidate their session.
3. Where a logout function is provided, test its effectiveness. After logging out, attempt to resuse the old token and determine whether it is still valid. If so, users remain vulnerable to some session hijacking attacks even after they have "logged out". You can use Burp to test this, by selecting a recent session-dependent request from the proxy history and sending it to Repeater to reissue after you have logged out from the application.
## Client Exposure to Token Hijacking:
1. Identify any XSS vulnerabilites within the application, and determine whether these can be exploited to capture the session tokens of other users (see Chapter 12)
2. If the application issues session tokens to unauthenticated users, obtain a token and perform a login. If the application does not issue a fresh token *following* a successful login, it is vulnerable to session fixation.
3. Even if the application does not issue session tokens to unauthenticated users, obtain a token by logging in, and then return to the login page. If the application is willing to return this page even though you are already authenticated, submit another login as a different user using the same token. If the application does not issue a fresh token after the second login, it is vulnerable to session fixation.
4. Identify the format of session tokens used by the application. Modify your token to an invented value that is validly formed, and attempt to log in. If the application allows you to create an authenticated session using an invented token, it is vulnerable to session fixation.
5. If the application does not support login, but processes sensitive user information (such as personal and payment details), and allows this to be displayed after submission(such as on a "verify my order" page), carry out the previous three tests in relation to the pages displaying sensitive data. If a token set during anonymous usage of the application can later be used to retrieve sensitive user information, the application is vulnerable to session fixation.
6. If the application uses HTTP cookies to transmit session tokens, it may be vulnerable to Cross-Site Request Forgery. First, log in to the application. Then confirm that a request made to the application but orginating from a page of a different application results in submission of the user's token. (This submission needs to be made from a window of the same browser process that was used to log in to the target application.) Attempt to identify any sensitive application functions whose parameters an attacker can determine in advance, and exploit this to carry out unauthorized actions within the security context of a target user. See Chapter 13 for more details on CSRF.
## Liberal Cookie Scope:
Review all the cookies issued by the application, and check for any `domain` attributes used to control the scope of the cookies.
1. If an application explicityly liberalizes its cookies' scope to a parent domain, it may be leaving itself vulnerable to attacks via other web applications.
2. If an application sets its cookies' domain scope to its own domain name (or does not specify a domain attribute), it may still be exposed to applications or functionality accesible via subdomains.
Identify all the possible domain names that will receive the cookies issued by the application. Establish whether any other web application or functionality is accessible via these domain names that you may be able to leverage to obtain the cookies issued to users of the target application.



















































































































