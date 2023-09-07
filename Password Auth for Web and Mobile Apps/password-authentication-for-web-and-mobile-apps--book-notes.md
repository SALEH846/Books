# Book -- Password authentication for web and mobile apps
**Author -- Dmitry Chestnykh**

### Basics
- Cryptography
	- User auth with passwords requires only two cryptographic things: cryptographically secure random number generator and password hashing function
	- The number of bugs grow with the number of lines of code
	- Generating quality randomness is vital for security
	- Don't overuse cryptography -- and make choice only after you know what you are doing
- Randomness
	- What is a good random number?
		- Unpredictable -- nobody can guess what the next bit will be even after knowing all the previous bits -- nobody can guess previous bits even if they see future bits -- i.e. the chance of guessing a bit is not greater than random, 50%.
		- Uniform distribution
	- Insecure random number generators
		- JavaScript, Node.js --> Math.random --> insecure
		- Python --> random module
	- Secure random number generators
		- JavaScript --> window.crypto.getRandomValues
		- Node.js --> crypto.randomBytes
	- Secure randomness is good but you still need enough of it. Don't use fewer than 16 bytes (which is 128 bits of entropy) for keys, tokens or identifiers
- Modulo bias
	- We generate random bytes and then encode them either using Hex or Base32 or Base64 -- there is no problem in it
	- But sometimes, we want strings that only contain characters, or only numbers. That's where people make most common mistake, they introduce modulo bias.
	- You get a number from a randomness generator and then use the modulo operation to fit it into the required range.
	- Example: n = crypto.randomBytes(1)[0] % 64 --> Got one byte (a number in the range 0-255) and then take modulo 64 of that number to get that number in the range from 0 to 64 from it. This example doesn't have modulo bias, because the modulo is a power of two.
	- Example: n = crypto.randomBytes(1)[0] % 10 --> incorrect --> the result will not be uniformly distributed from 0 to 10 -- some numbers will be more likely to appear in the result than others.
- UUID
	- Also known as GUID.
	- The only good version is v4, generated using a secure randomness generator.
	- Example: 7968f9b3-51d1-4d03-bd1a-ed13913fe0a6
	- Hex characters split with dashes -- 16 bytes -- however, they don't contain 128 bits of entropy -- some bits are reserved for version and other silly stuff -- the '4' you see in third group is a version number and other two or three bits are taken to indicate the 'variant' -- UUIDv4 will contain 121 or 122 bits of randomness -- However, they are still globally unique and unpredictable.
	- Use only UUIDv4.
	- Use a UUID generator that gets entropy from a secure randomness generator
- Constant time comprison
	- Don't use taditional methods of string comparison which compare the corresponding bytes in each string such as `==` and others -- they will return as soon as they find first mismatching bytes -- they will first compare the lengths of both the strings and if do not match they will immediately return false -- this makes the system vulnerable to timing attacks.
	- Always use contant time comparisons when comparing passwords.
	- JavaScript --> @statelib/constant-time
	- Node.js --> crypto.timingSafeEqual
	- Python --> hmac.compare_digest
	- The timing problem arises not only when using a comparison function -- consider a database index, when the database engine looks up the string -- this is why you should not use secret tokens or identifiers as database keys if the inability to guess them is the only thing that makes them secure -- when the database lookup is not constant time and depends on the bytes of the lookup string, the atatcker can perform a timing attack and discover existing keys -- To avoid this error, use split tokens: where one part is a lookup string and another part is a string that will be compared on the contant time.
- Audit logging and reporting
	- Following actions concerning user accounts should be logged
		- Successful or unsuccessful login attempts
		- Changes to usernames and emails
		- Changes to passwords
		- Password reset attempts
		- Multi-factor authentication set ups and use attempts
	- You should not log passwords, password hashes, the verification part of sessions or confirmation tokens, or secret keys (including two-factor authentication).
	- May want to notify users about successful login attempts and changes to thier account by email
	
### Users
- Should be case-insensitive --> (John, john and jOhN are all same)
- can be choosen by users or randomly choosen by the system
- Email are used for verification
- Allowing users to use usernames instead of email when authenticating is not the same thing -- using usernames allow pseudonymity
- Important paragraph related to usernames vs emails --> page: 17
- Filtering usernames:
	- Disallow profanities in usernames
	- People with malicious intentions will try to create accounts with usernames that look like official usernames such as "admin" or your company name
	- For this, you will need a list of bad usernames
	- User's are inventive, they will replace o's with zeros, use greek alphabet for some letters -- the solution is to normalize the usernames before checking the list
- Validating email addresses
	- The only way is to send confirmation link and wait for it to be clicked
	- RegEx that will validate the 99% of the email addresses in the world -- [a-z0-9!#$%&'*+/=?^_`\{|}~.-]+@[a-z0-9-]+(.[a-z0-9-]+)*$/gi
	- Use punicode variant of the email addresses which contain non-ASCII characters
- Confirming email addresses
	- Should be confirmed to ensure that they belong to the user
	- Also perform confirmation when user change email address
	- Hackers can create multiple accounts for emails that they do not own. So, when the actual user tries to create an account they will be turned away. Solution is that if the emails stays unconfirmed for day or two, delete it.
	- When changing the email addresses, only replace the old with new one after it is being confirmed
- Universal Confirmation System
	- Basic structure of confirmations table in the database -- page: 24
	- Very important section -- page: 24-27
- Changing usernames
	- Don't release old usernames to other users (but allow the user who changed it to go back to the old one)
- Changing email addresses
	- Only after the user has confirmed the new email by clicking on the link that you sent him in email, replace the old email with the new one
- Requiring re-authentication
	- Important actions with user accounts, such as changing password or email, must require re-authentication.
- Unicode issues
	- Important section -- page: 29

### Passwords
- Password quality control
	- Miminum length -- at least 10 characters
	- use an algorithm that catches passwords that are too simple
	- library for checking password quality - passwdqc -- another library - zxcvbn
	- Don't use third-party services to check for leaked passwords
- Characters and emojis in passwords
	- How to Normalize passwords -- NIST recommended standards: NFKC, NFKD
	- Emojis in passwords
- Password hashing
	- Common mistakes -- storing passwords as plain text, storing the passwords after encrypting them or using bad hash function
	- Password hashing functions take three inputs:
		1. password -- password bytes (encoded in UTF-8, by convention)
		2. salt -- array of random bytes (usually 16 or 32 bytes)
		3. cost -- configurable parameter that defines how much time or compute resources it takes to produce the output
	- A rule of thumb for password hashing: use the fastest implementation of slowest algorithm
	- Common password hashing functions are:
		1. PBKDF2 -- stands for Password-Based Key Derivation Function version Two -- weakest of all - cost parameter is called "rounds" or "iterations" -- tune iterations so that it takes at least 200ms to produce output
		2. bcrypt -- cost parameter relates to the number of internal rounds, it is the power of 2 (cost 12 means 2^12 rounds), so computational cost grows exponentially -- the amount of memory it takes for computation is fixed and cannot be configured -- used by GitHub and Twitter -- the result is the hash encoded along with the salt and cost parameter -- uses 16-byte salts -- maximum password length is 72 bytes -- adjust the cost so that it takes at least 200ms to compute the hash
		3. scrypt -- does CPU-consuming computations and also uses a specified amount of memory to do computations, using more memory increases the atatcker's cost and makes parallel attacks expensive -- Scrypt is a sequentia memory-hard function which means that its workload cannot be easily split into parts that use fewer amounts of memory and run in parallel
			- Scrypt accepts password bytes, salt bytes and three parameters: N, r and p and returns the requested amount of bytes as a hash.
			- Scrypt output is a uniformly distributed pseudorandom value, thus to reproduce the result for verification, you should store the salt and the cost parameters.
			- Don't use a single global configuration for N, r and p; rather store them for each user in the database.
			- p: parallelization parameter -- how many threads to use for computing the result
			- r: block size parameter of the internal function -- should be set to 8 for the currently available processors
			- N: memory parameter -- must be a power of 2 -- N is not alone in configuring RAM consumption: r and p also influence it
			- The number of bytes of memory scrypt will use is roughly described with this formula:
				memory = 128 * N * r * p
			- Read more about how to configure this formula
		 	- yescrypt -- modified version of scrypt -- better and easy to use
		4. Argon2 -- both memory and computationally hard -- unlike scrypt it can be separately configured for how much memory it consumes and how much time the computation takes.
			- Three variants:
				1. Argon2i
				2. Argon2d
				3. Argon2id
			- For server-based user authentication, use Argon2id
			- You should not use Argon2 implementation written in pure JavaScript or another scripting language -- since JavaScript doesn't have optimized 64-bit arithmetic or 32-bit integer multiplications
			- Argon2 cost parameters are the memory cost in kilobytes, the number of iterations it should go over the memory, and the parallelism
- GoodBye, rainbow tables
	- Rainbow tables are nothing to worry about with password hashing functions because they include salt.
- Prehashing for scrypt and bcrypt
	- Prehashing is a trick to protect against bcrypt's and scrypt's long password issues.
	- Instead of directly hashing passwords with a password hashing function, first, we prehash the password with a fast cryptographic hash function to turn it into a fixed-length short string and then use the result as a password input for the password hashing function.
- Peppering and encrypting keys
	- Peppering is an additional protection for password hashes that makes them impossible to crack without knowing the secret key. It works by adding a secret input, called "pepper", to the hashing function.
	- It is kept in the memory of the server that does authentication.
	- It provides protection for cases when the attacker can access the password hashes database, but doesn't have access to the application server.
	- Don't store pepper in the database. Keep it on the application server.
	- Pepper is not per-user, but a global value for all users.
	- Use prehashing in the case of bcrypt, because it has a password length problem.
		`prehash = HMAC-SHA-256(pepper, password)`
		`hash = bcrypt(prehash + salt, cost)`
	- For PBKDF2, scrypt and yescrypt:
		`newSalt = CONCAT(pepper, salt)`
		`hash = scrypt(password, newSalt, N, r, p)`
	- For Argon2, just pass the pepper into the secret key input.
	- Another way to introduce a secret key into the scheme is to encrypt the hash.
		`hash = scrypt(password, salt, N, r, p)`
		`encryptedHash = ENCRYPT(secretKey, hash)`
- Hash length
	- PBKDF2, scrypt and Argon2 allow you to request a practically unlimited number of output bytes.
	- The hash should be long enough to not cause a collision with another hash (which will simplify dictionary attacks and allow one user to authenticate with a different user's password).
	- It shouldn't be too long since there is no more security to be gained after some length.
	- 16 to 32 bytes is enough.
	- With bcrypt, you don't control the hash output length, so stick with whatever output it gives you (which also includes identifier, version, salt, and the 23-byte hash).
- Encoding and storing password hashes
	- In some cases, you have to store the parameters in separate columns in the database like in the case you are scrypt or PBKDF2.
	- But in the case of bcrypt, everything is encoded in the hash of the password.
- Hashing upgrades
	- What to do when updating cost parameters
- Client-side password prehashing
	- For most cases, the recommendation is against any client-side processing. If you still want it, Read that section of the book.
- Resetting passwords
	- The most common way is to use email.
	- Users ask to reset their password, we send them a link, we pretend we delivered the email secureky and that it is them, the original user, and not someone who hacked into theor email, is now trying to reset their password.
	- How to implement password resets?
		- Your service presents a form where the user can enter their email address.
		- If the user with this email address doesn't exist in the system, simply don't send anything to anyone. If it does, send the password reset instructions that include the link.
		- To create a reset link, first, mark the account as being in the state of password reset, and a generate a random token: a 16-byte identifier and a 16-byte verifier stored in the hashed form; combine the token into a single string and put it into the link.
		- Make sure this token has a short expriration period (half an hour would be plenty).
		- We can allow a single user to have many outstanding password resets. If they request the password reset twice without acting on any of them, they should be able to click the link in any of the messages (at least, until they expire).
		- However, when the user sets a new passord or when they successfully login with the old password before they clicked the link, make sure to delete of theor reset tokens.
		- After they set a new password, delete all of the sessions except for the current one.
		- For detailed algorithm --> `page: 61`
- Changing passwords
	- To change the password, the user must be logged in.
	- And when changing the password, they must also enter their current password.
	- All the sessions and password reset requests must be invalidated when the password is changed.
- Against enforced password rotation
	- Enfore periodical password rotation was once popular, and, unfortunaltely, is still prcaticed. Every month or so user are asked to change their passwords. This is a bad practice that should be stopped.
	- Only enforce password rotation when you discover a password leak.
- Secret questions
	- Detail --> `page: 64`
	- They are controversial in security community.
	- Secret questions won't help with targeted attacks but can be useful as an additional measure to slow down attackers in some instances. Again, an additional measure, not the only one! You still have to use email for verification.
	- Do not ask questions the answers to which are easilt found. Ask questions that are truly personal. Questions, of course, should be inclusive and not offensive.
	- Aswers to questions are low-entropy secret data. What other secret data is usually low.entropy? Passwords. Do, protect answers in the same ay as passwords: do not store them as plaintext, instead, hash them with a passord hashing function, with a random salt.
	- It's not easy for users to remember how they typed theor answers when they suddlenly need to re-type them a few years later, so you need to normalize them. Hash the normalized answers with the password hashing function and store the results. To verofy, normalize the answer entered by the user, hash it, and then compare the hash with the stored one.
	- In the password reset flow, ask secret questions after the user clicks on the reset link in the email message, not before, to prevent anyone else from guessing the answers.

### Multi-factor authentication
- What is multi-factor authentication?
	- It is an additional measure to protect accounts in case of a user's password leak.
	- It requires a secondary secure channel or device.
- Since the codes are typed by users, just like passwords, they are vulnerable ti phishing.
- The most used standard for one-time code authentication is TOTP (Time-based One-Time Password).
- There is a difference between two-step and two-factor.
	- Two-factor require the second factor to be present -- usually a device, such as a smartphone or a security key.
	- Two-step can use email, SMS, etc. They are not the second thing.
- Rate limiting
	- Rate limiting is very important for two-factor authentication with codes.
	- Since two-factor authentication happens after the user (or the attacker) correctly entered their password, your server knows what account to apply the rate limiting to, thus you should record the attempts for each user, and reset the count on successful authentication. Since the limits are applied to the user account, not to the IP, this makes parallel guesses infeasible.
	- Apply exponential time limits for re-trying after failed login attempts.
	- After a few failed code attempts, lock the account. 
	- Read more in the book.
- TOTP (Time-based One-Time Password)
	- Detail in the book --> `page: 69`
- SMS --> `page: 71`
- Security keys: U2F, WebAuthn
	- U2F is dead, forget about it.
	- WebAuthn is the nre standard.