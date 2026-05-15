---
title: "Webverselabs Calliope Gallery Challenge File Upload  "
date: 2026-05-15 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Medium , Challenge , Web_Attacks ]
---


## Scenario : 

Calliope was founded in 2021 by two former MFA students who couldn't get a toehold on the New York gallery circuit and decided to build their own. Three years on, the roster is about eighty emerging painters, illustrators and printmakers across North America, with bricks-and- mortar viewing rooms in Tribeca and Highland Park. Last spring a contractor wired in a thumbnail-resizer to keep the portfolio dirs fast on mobile and left a small config tweak in the upload tree to make the resizer work; the team has long since forgotten the integration is in there at all.


## Solution :

We first visit the webapp like a normal user to see different endpoints , tried Fuzzing for files and directories but that didn't return anything useful .

<img width="1394" height="880" alt="image" src="https://github.com/user-attachments/assets/6b8aa460-f3f2-450c-be8b-80a652ed30f3" />

We created an account , and we see that we are able to upload Images to the web app , as long as it is JPEG . 

<img width="1408" height="828" alt="image" src="https://github.com/user-attachments/assets/ba2238b5-4462-4181-b253-a075802e74bf" />

Now let's try to upload an image then change it to a php file to see if we can bypass the front end valdiation . 

<img width="1294" height="710" alt="image" src="https://github.com/user-attachments/assets/4949d480-72b1-4aa8-84d2-b034159693e8" />

Now on Burp , we can modify the file type to see if this bypasses anything . 

<img width="1496" height="840" alt="image" src="https://github.com/user-attachments/assets/8ac64734-62e7-44d8-88ae-5bf83b3990bd" />

This didn't work which means there is probably some sort of filtering on the server side as well , now what i usually do is add magic Bytes at the beguinning of the file which tells the server that it is an actual JPEG image from the first Bytes .

For JPG the magic byte will be FF D8 FF E0 in HEX .


```bash
FF D8 FF E0  : hex representation .

==> To interpret this as a binary :

\xff\xd8\xff\xe0 : (\x prefix tells the shell "this is hex, interpret as binary")

```

For example if we add our payload to a file and name it shell2.jpg , the server will know it's not an image by checking the first few bytes : 

<img width="387" height="291" alt="image" src="https://github.com/user-attachments/assets/1ae2d407-f339-4bb3-86fe-501784bb11c9" />

Now if we add the Magic bytes at the beguinning : 

```bash
echo -e '\xff\xd8\xff\xe0aaaaa' > shell2.jpg (make sure you add the -e )

echo '\xff'    ==> sees \xff as 4 characters: \  x  f  f
echo -e '\xff' ==> sees \xff and thinks "oh \x means HEX" → writes the actual byte FF

```

<img width="561" height="281" alt="image" src="https://github.com/user-attachments/assets/b4b3d62e-63aa-40b7-9c75-61fddbfabf71" />


Now let's try to upload this one to see : 


<img width="1532" height="703" alt="image" src="https://github.com/user-attachments/assets/e67237c5-f90d-4a74-9fd0-89eff64ef0bd" />


Perfect it worked , now let's try to write a PHP one liner to see if we can execute PHP code like this : 


<img width="781" height="271" alt="image" src="https://github.com/user-attachments/assets/4878f1bf-e43e-48ec-9ab3-8055f955e80a" />


Now we upload the Shell.jpg without any issues , But to trigger the Web Shell we need to first find where the server stores it since this is all we get after uploading it : 


<img width="971" height="785" alt="image" src="https://github.com/user-attachments/assets/baeb632c-b0a0-4182-ada8-217f7ab9f7cf" />


Now if we check our Burp History , After we see the POST request made to the server (when we hit Submit) , we find this GET request . 


<img width="1414" height="685" alt="image" src="https://github.com/user-attachments/assets/ab4e4f9c-d850-4d72-b980-005ccd314bb2" />


Perfect now let's try and visit the Web Shell we just downloaded . 


<img width="1144" height="431" alt="image" src="https://github.com/user-attachments/assets/9f371074-acbb-4da6-8be4-8435d9f45153" />


The flag is located in /flag.txt .


<img width="1168" height="315" alt="image" src="https://github.com/user-attachments/assets/68286816-5971-4aa4-965c-99f94a6e902b" />


Perfect Now we can use this to get a Reverse Shell , escalate to root , pivot to the internal network ... , but that's beyond the scope for this challenge . 



