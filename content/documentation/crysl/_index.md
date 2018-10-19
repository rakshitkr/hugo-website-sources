---
title: "The CrySL Language"
date: 2018-09-03T19:48:11+02:00
---

The [static analysis](/cognicrypt/documentation/codeanalysis) is based on `CrySL rules` that specify the *correct* use of an application programming interface (API). `CrySL` is a domain-specific language that allows to specify usage patterns of APIs. The static analysis reports any deviations from the usage pattern defined within the rules. 

## Syntax of the Domain-Specific Language CrySL

CogniCrypt's error markers are generated based on violations of *rules*. Rules in CogniCrypt are written in `CrySL`. `CrySL` is a domain-specific language for the specification of correct cryptograhy API uses in Java. The Eclipse plugin CogniCrypt ships with a XText editor that supports the `CrySL` syntax. `CrySL` encodes a white-list approach and specifies how to *correctly* use crypto APIs. We discuss some of the most important concepts of the rule language here, the [research paper](http://drops.dagstuhl.de/opus/volltexte/2018/9215/pdf/LIPIcs-ECOOP-2018-10.pdf) provides more detailed insides on the language. 

CogniCrypt ships with a default rule set for the [Java Cryptographic Architecture (JCA)](https://docs.oracle.com/javase/8/docs/technotes/guides/security/crypto/CryptoSpec.html). [This page](/cognicrypt/documentation/codeanalysis) provides details on the standard rule sets.


Each `CrySL` rule is a specification of a single Java class. A short example of a `CrySL` rule for `javax.crypto.Cipher` is shown below. 

```
SPEC 	javax.crypto.Cipher
OBJECTS 
	java.lang.String trans;
	byte[] plainText; 
	java.security.Key key;
	byte[] cipherText;
EVENTS 
	Get: getInstance(trans); 
	Init: init(encmode, key); 
	doFinal: cipherText = doFinal(plainText); 
ORDER 
	Get, Init, (doFinal)+ 
CONSTRAINTS  
	encmode in {1,2,3,4};
	part(0, "/", trans) in {"AES", "Blowfish", "DESede", ..., "RSA"};
	part(0, "/", trans) in {"AES"} => part(1, "/", trans) in {"CBC"};
REQUIRES 
	generatedKey[key, part(0, "/", trans)];
ENSURES 
	encrypted[cipherText, plainText]; 
```
Each rule has a `SPEC` clause that lists the fully qualified class name the following specification holds for (in this case `javax.crypto.Cipher`)
The `SPEC` clause is followed by the blocks `OBJECTS`, `EVENTS`, `ORDER`, `CONSTRAINTS`, `REQUIRES` and `ENSURES`. 
Within the `CONSTRAINTS` block each rule lists `Integer` and `String` constraints.  The `OBJECTS` clause lists all variable names that can be used within all blocks of the rule. The `EVENTS` block lists API method calls that can be made on each `Cipher` object. When an event is encountered, the actual values of the events parameters are assigned to respective variable name listed in the rule. These parameter values can then be constrained by `CONSTRAINTS`. 

### The CONSTRAINTS section

The `Cipher` rule lists `encmode in {1,2,3,4};` within its `CONSTRAINTS` block. The value `encmode` that is passed to method `init(encmode, cert)` is restricted to be one of the four integers. In other terms, whenever the function `init` is called, the value passed in as first parameter must be in the respective set.  The constraint `part(0, "/", trans) in {"AES", "Blowfish", "DESede", ..., "RSA"}`  refers to the fact that at the call to `Cipher.getInstance(trans)` the `String trans` must be correctly formed. The function `part(0, "/", trans)` splits the `String` at the character `"/"` and returns the first part. Hence the constraint restricts the first part prior of any `"/"` to be either `"AES"` or `"RSA"`. The third constraint (`part(0, "/", trans) in {"AES"} => part(1, "/", trans) in {"CBC"};`) is a conditional constraint: If the first part of `trans` is `"AES"`, then the second part of `trans` must be `"CBC"`. For example, this conditional rule warns a developer writing `Cipher.getInstance("AES/ECB/PKCS5Padding")` instead of `Cipher.getInstance("AES/CBC/PKCS5Padding")`. 

### The ORDER section

The `ORDER` section of a rule specifies a regular-expression like description of the excepted events to occur on each individual object. For the `Cipher` rule the order is `Get, Init, (doFinal)+`. The terms `Get`, `Init` and `doFinal` are labels and group a set of API methods that are defined within the `EVENTS` block. The regular expression stated in the `ORDER` section enforces the following order on a `Cipher` object: The object must be create by a `Get`, i.e., `Cipher.getInstance`, call, then `Init` must be called before eventually any (and at least one) times the method `doFinal` is called. A programmer who writes the program below contradicts the `ORDER` section of the `CrySL` rule: A call to `init` on the `cipher` object is missing between the `getInstance` and `doFinal` call (the missing call is commented out).

```
Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding");
//cipher.init(Cipher.ENCRYPT_MODE, secretKeySpec);
cipher.doFinal(plainText);
```

### The ENSURES and the REQUIRES section

Cryptographic tasks are more complex and involve interaction of multiple object instances at runtime. For example for an encryption task with a `Cipher` instance, the `Cipher` object must be initialized with a securely generated key. The API of the `Cipher` object has a method `init(encmode,key)` where the second parameter is the respective key and is of type `SecretKeySpec`. For a correct use of the `Cipher` object, the key must be used correctly as well.

To cope with these object interactions, `CrySL` allows the specification of what we call *predicates* that are listed in the blocks `REQUIRES` and `ENSURES`. An object that is used in coherence with the rule receives the predicate listed in the `ENSURES` block. In turn, the block `REQUIRES` allows rules to force other objects to hold certain predicates.

The specification of the `Cipher` rule lists a predicate `generatedKey[key,...]` within its `REQUIRES` block. The variable name `key` refers to the same object that is used within the event `Init: init(encmode, key);` of the  `EVENTS` block. Hence, the key object must receive this predicate which is listed in the rule for `javax.crypto.SecretKeySpec`. 

```
SPEC javax.crypto.spec.SecretKeySpec
OBJECTS
	java.lang.String alg;
	byte[] keyMaterial;
		
EVENTS
	c1: SecretKeySpec(keyMaterial, alg);
...
REQUIRES
	randomized[keyMaterial]; 
ENSURES
	generatedKey[this, alg];
```

Above is an excerpt of the rule for `SecretKeySpec`. The predicate `generatedKey` is listed within the `ENSURES` block of this rule. The static analysis labels any object of type `SecretKeySpec` by `generatedKey` when the analysis finds the object to be used correctly (with respect to its `CrySL` rule).
