[![Build Status](https://travis-ci.org/jwtk/jjwt.svg?branch=master)](https://travis-ci.org/jwtk/jjwt)

# Java JWT: JSON Web Token for Java

JJWT aims to be the easiest to use and understand library for creating and verifying JSON Web Tokens (JWTs) on the JVM.

JJWT is a 'clean room' implementation based solely on the [JWT](https://tools.ietf.org/html/draft-ietf-oauth-json-web-token-25), [JWS](https://tools.ietf.org/html/draft-ietf-jose-json-web-signature-31), [JWE](https://tools.ietf.org/html/draft-ietf-jose-json-web-encryption-31) and [JWA](https://tools.ietf.org/html/draft-ietf-jose-json-web-algorithms-31) RFC draft specifications.

## Installation

Use your favorite Maven-compatible build tool to pull the dependency (and its transitive dependencies) from Maven Central:

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>0.2</version>
</dependency>
```

Note: JJWT depends on Jackson 2.x.  If you're already using an older version of Jackson in your app, [read this](#olderJackson)

## Usage

Most complexity is hidden behind a convenient and readable builder-based [fluent interface](http://en.wikipedia.org/wiki/Fluent_interface), great for relying on IDE auto-completion to write code quickly.  Here's an example:

```java
import io.jsonwebtoken.Jwts;
import static io.jsonwebtoken.SignatureAlgorithm.*;

//Let's create a random signing key for testing:
Random random = new SecureRandom();
byte[] key = new byte[64];
random.nextBytes(key);

String compact = Jwts.builder().setSubject("Joe").signWith(HS256, key).compact();
```

How easy was that!?

Now let's verify the JWT (you should always discard JWTs that don't match an expected signature):

```java
Claims claims = Jwts.parser().setSigningKey(key).parseClaimsJws(compact).getBody();

assert claims.getSubject().equals("Joe");
```

You have to love one-line code snippets in Java!

But what if signature validation failed?  You can catch `SignatureException` and react accordingly:

```java
try {

    Jwts.parser().setSigningKey(key).parse(compactJwt);

    //OK, we can trust this JWT

} catch (SignatureException e) {

    //don't trust the JWT!
}
```

## Supported Features

* Creating and parsing plaintext compact JWTs

* Creating, parsing and verifying digitally signed compact JWTs (aka JWSs) with the following algorithms:
    * HS256: HMAC using SHA-384
    * HS384: HMAC using SHA-384
    * HS512: HMAC using SHA-512
    * RS256: RSASSA-PKCS-v1_5 using SHA-256
    * RS384: RSASSA-PKCS-v1_5 using SHA-384
    * RS512: RSASSA-PKCS-v1_5 using SHA-512
    * PS256: RSASSA-PSS using SHA-256 and MGF1 with SHA-256
    * PS384: RSASSA-PSS using SHA-384 and MGF1 with SHA-384
    * PS512: RSASSA-PSS using SHA-512 and MGF1 with SHA-512

## Currently Unsupported Features

* [Non-compact](https://tools.ietf.org/html/draft-ietf-jose-json-web-signature-31#section-7.2) serialization and parsing.
* Elliptic Curve signature algorithms `ES256`, `ES384` and `ES512`.
* JWE (Encryption for JWT)

These feature sets will be implemented in a future release when possible.  Community contributions are welcome!

## Release Notes

### 0.2

#### More convenient Claims building

This release adds convenience methods to the `JwtBuilder` interface so you can set claims directly on the builder without having to create a separate Claims instance/builder, reducing the amount of code you have to write.  For example, this:

```java
Claims claims = Jwts.claims().setSubject("Joe");

String compactJwt = Jwts.builder().setClaims(claims).signWith(HS256, key).compact();
```

can now be written as:

```java
String compactJwt = Jwts.builder().setSubject("Joe").signWith(HS256, key).compact();
```

A Claims instance based on the specified claims will be created and set as the JWT's payload automatically.

#### Type-safe handling for JWT and JWS with generics

The following < 0.2 code produced a JWT as expected:

```java
Jwt jwt = Jwts.parser().setSigningKey(key).parse(compact);
```

But you couldn't easily determine if the `jwt` was a `JWT` or `JWS` instance or if the body was a `Claims` instance or a plaintext `String` without resorting to a bunch of yucky `instanceof` checks.  In 0.2, we introduce the `JwtHandler` when you don't know the exact format of the compact JWT string ahead of time, and parsing convenience methods when you do.

##### JwtHandler

If you do not know the format of the compact JWT string at the time you try to parse it, you can determine what type it is after parsing by providing a `JwtHandler` instance to the `JwtParser` with the new `parse(String compactJwt, JwtHandler handler)` method.  For example:

```java
T returnVal = Jwts.parser().setSigningKey(key).parse(compact, new JwtHandler<T>() {
    @Override
    public Object onPlaintextJwt(Jwt<Header, String> jwt) {
        //the JWT parsed was an unsigned plaintext JWT
        //inspect it, then return an instance of T (see returnVal above)
    }
    
    @Override
    public Object onClaimsJwt(Jwt<Header, Claims> jwt) {
        //the JWT parsed was an unsigned Claims JWT
        //inspect it, then return an instance of T (see returnVal above)
    }

    @Override
    public Object onPlaintextJws(Jws<String> jws) {
        //the JWT parsed was a signed plaintext JWS
        //inspect it, then return an instance of T (see returnVal above)
    }

    @Override
    public Object onClaimsJws(Jws<Claims> jws) {
        //the JWT parsed was a signed Claims JWS
        //inspect it, then return an instance of T (see returnVal above)
    }
});
```

Of course, if you know you'll only have to parse a subset of the above, you can use the `JwtHandlerAdapter` and implement only the methods you need.  For example:

```java
T returnVal = Jwts.parser().setSigningKey(key).parse(plaintextJwt, new JwtHandlerAdapter<Jwt<Header, T>>() {
    @Override
    public Object onPlaintextJws(Jws<String> jws) {
        //the JWT parsed was a signed plaintext JWS
        //inspect it, then return an instance of T (see returnVal above)
    }

    @Override
    public Object onClaimsJws(Jws<Claims> jws) {
        //the JWT parsed was a signed Claims JWS
        //inspect it, then return an instance of T (see returnVal above)
    }
});
```

##### Known Type convenience parse methods

If, unlike above, you are confident of the compact string format and know which type of JWT or JWS it will produce, you can just use one of the 4 new convenience parsing methods to get exactly the type of JWT or JWS you know exists.  For example:

```java

//for a known plaintext jwt string:
Jwt<Header,String> jwt = Jwts.parser().parsePlaintextJwt(compact);

//for a known Claims JWT string:
Jwt<Header,Claims> jwt = Jwts.parser().parseClaimsJwt(compact);

//for a known signed plaintext JWT (aka a plaintext JWS):
Jws<String> jws = Jwts.parser().setSigningKey(key).parsePlaintextJws(compact);

//for a known signed Claims JWT (aka a Claims JWS):
Jws<Claims> jws = Jwts.parser().setSigningKey(key).parseClaimsJws(compact);

```

<a name="olderJackson"></a>
#### Already using an older Jackson dependency?

JJWT depends on Jackson 2.4.x (or later).  If you are already using a Jackson version in your own application less than 2.x, for example 1.9.x, you will likely see [runtime errors](https://github.com/jwtk/jjwt/issues/1).  To avoid this, you should change your project build configuration to explicitly point to a 2.x version of Jackson.  For example:

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.4.2</version>
</dependency>
```
